---
layout: post
title: How I Unwillingly Wrote a Web Server
---

It feels like [every](https://dev.twitch.tv/docs/authentication/) [single](https://developer.spotify.com/documentation/web-api/tutorials/code-pkce-flow) [web](https://discord.com/developers/docs/topics/oauth2) [service](https://developer.x.com/en/docs/authentication/overview) wants to use OAuth to get user data, and for good reason too. The alternatives include requiring users to create access tokens manually (yikes) and asking users to input their usernames/passwords to impersonate them (giga yikes). At least with OAuth, users get to see a pretty screen with a big "Approve" button.

The OAuth spec defines [many different auth flows](https://auth0.com/docs/get-started/authentication-and-authorization-flow/which-oauth-2-0-flow-should-i-use), but I want to focus specifically on Authorization Code with Proof Key for Code Exchange, or PKCE for short. In summary, the steps for PKCE are:

1. App creates a random `code_verifier` string and associated `code_challenge` string
2. App redirects the user to the `/authorize` endpoint of the Authorization Server
3. User completes authorization on the UI
4. **Authorization Server redirects to application, providing an authorization `code`**
5. App calls `/token` endpoint of the Authorization Server, passing in the authorization `code` and `code_challenge` (from step 1)
6. Authorization Server responds with an access and (possibly) refresh token
7. App uses the access token to call APIs

If you've implemented OAuth on a web app, you've likely already done all of this (though maybe through a library that hides all the inner workings). Native applications run into an issue on step 4, though: _how do you redirect from a website to a desktop application?_

## Custom URI Schemes ("Deep Links")

The extremely platform-dependent answer is to register a custom URI scheme for your application, so something like `my-cool-app:callback` opens your application with the appropriate intent/parameters. The exact mechanism differs between platforms ([Intent Filters](https://developer.android.com/guide/components/intents-filters) on Android, [URL Types](https://developer.apple.com/documentation/xcode/defining-a-custom-url-scheme-for-your-app) on iOS/macOS, [Registry](<https://learn.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa767914(v=vs.85)>) on Windows), but once you have it set up, the OS deals with launching/notifying your app.

```rust
use winreg::{enums::HKEY_CURRENT_USER, RegKey};

fn register_app_scheme() {
    let scheme = String::from("my-cool-app");
    let hkcu = RegKey::predef(HKEY_CURRENT_USER);
    let base = Path::new("Software").join("Classes").join(&scheme);

    let exe = std::env::current_exe()
        .unwrap()
        .display()
        .to_string()
        .replace("\\\\?\\", "");

    let (key, _) = hkcu.create_subkey(&base).unwrap();
    key.set_value("", &format!("URL:{}", scheme))
        .unwrap();
    key.set_value("URL Protocol", &"").unwrap();

    let (icon, _) = hkcu.create_subkey(base.join("DefaultIcon")).unwrap();
    icon.set_value("", &format!("{},0", &exe)).unwrap();

    let (cmd, _) = hkcu
        .create_subkey(base.join("shell").join("open").join("command"))
        .unwrap();

    cmd.set_value("", &format!("\"{}\" --uri \"%1\"", &exe))
        .unwrap();

    println!(
        "[custom_scheme_handler] Registered URI scheme `{}:`",
        scheme
    );
}
```

_Register `my-cool-app:` to launch the application with `--uri <path>` as a CLI flag on Windows._

This is still not a complete answer though, since on Windows (and maybe macOS?), this would launch a second instance of your application, so you'll also have to deal with setting up a mutex for single-instance behavior and an IPC channel to notify the original instance that the callback was hit.

Of course, it's likely that a lot of this lower-level plumbing has already been abstracted away for you by a library/SDK.

## Transient Localhost Server (uh oh)

Say I didn't want to deal with operating systems and their weird URL APIs.

Also say I didn't want to use libraries.

Also say I was using C.

Well, if you break the redirection problem down, all we really need is a URL to our app, right? So let's just... make one.

### What's in an HTTP server, anyway?

_For the scope of this article, we're only going to deal with HTTP/1.1._

HTTP is a shockingly simple protocol when you really think about it.

1. Client opens a TCP connection to server (normally port 80, but not necessarily)
2. Client sends an HTTP request (see below)
3. Server sends an HTTP response (see below)
4. Server closes connection\*

\*Technically, the socket can be left open and reused for future requests, bypassing the TCP handshake overhead. For our purposes though, we'll close it to keep things simple.

**One important thing to note is that HTTP uses `<CRLF>` (`\r\n`) as its line endings, not just `\n`.**

### Dissecting HTTP/1.1 Requests

[Specification](https://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html)

Here's a sample POST request as you might receive from a web browser:

```
(1) POST /lists HTTP/1.1
(2) Host: www.example.com
    Content-Type: application/json
    ... other headers ...
(3) <blank line>
(4) {
      "name": "My new list",
      "items": [
        "Wow look",
        "It's a list",
      ]
    }
```

1. Request line, in the format `<Method> <URL> <HTTP-Version>\r\n`
2. Headers, each in the format `<Key>: <Value>\r\n`
3. Blank line (`\r\n`) to signify the end of the headers
4. Request body in whatever encoding (JSON, XML, form multipart, binary)

- Naturally, `GET`, `HEAD`, and `OPTIONS` don't (at least, _shouldn't_) have request bodies.

You can grab one of these yourself by running a TCP server, then pointing a web browser to that port and looking at the received message. Here I'm using `nc` (the one that ships with macOS) to listen on port 1337.

```console
preyneyv:~ $ nc -l 1337
GET / HTTP/1.1
Host: localhost:1337
Connection: keep-alive
sec-ch-ua: "Chromium";v="128", "Not;A=Brand";v="24", "Google Chrome";v="128"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "macOS"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8

```

### Dissecting HTTP/1.1 Responses

[Specification](https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html)

An HTTP/1.1 response should look like this:

```
(1) HTTP/1.1 200 OK
(2) Content-Type: text/html
    ... other headers ...
(3) <blank line>
(4) <!DOCTYPE html>
    <html>
    <body>Hey look, a message!</body>
    </html>
```

1. Status line, in the format `<HTTP-Version> <Status-Code> <Reason>\r\n`
2. Headers, each in the format `<Key>: <Value>\r\n`
3. Blank line (`\r\n`) to signify the end of the headers
4. Response body in whatever encoding (HTML, JSON, XML, binary)

- Naturally, certain request types (`HEAD`) and response codes (`204 No Content`) shouldn't have a response body.

And that's literally all we need to know to make an HTTP server. To prove it, let's use `nc` (macOS version) again.

- Use `-c` to make it use CRLF instead of LF as its line endings.
- Make a request to `localhost:1337` from your browser. You'll see that it starts spinning, waiting for a response.
- In your terminal, type an HTTP response.
- Press `Ctrl+D` to close the socket.

(I'm using `->` to indicate the request the browser sends, and `<-` to indicate what you should type into your terminal.)

```console
preyneyv:~ $ nc -c -l 1337
-> GET / HTTP/1.1
-> Host: localhost:1337
-> ... other headers ...
->
<-  HTTP/1.1 200 OK
<-  Content-Type: text/html
<-
<-  hello! <b>this is cool!</b>
^D
```

If you did everything right, you should see your message in your browser!

### Building it in C

Let's start with a basic TCP server that accepts, then immediately closes a connection. If you've done socket programming in C (or even Python), this should be immediately familiar. If not, here's a solid [intro to network programming](https://beej.us/guide/bgnet/html/).

```c
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <unistd.h>

#define SERVER_PORT 1337

int main(void) {
  int server = socket(AF_INET, SOCK_STREAM, 0);

  int enable = 1;
  setsockopt(server, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));

  struct sockaddr_in address = {.sin_family = AF_INET,
                                .sin_addr = {.s_addr = htonl(INADDR_ANY)},
                                .sin_port = htons(SERVER_PORT)};
  if (bind(server, (struct sockaddr *)&address, sizeof(address)) == -1) {
    perror("bind");
    goto cleanup_failure;
  }

  if (listen(server, 5) == -1) {
    perror("listen");
    goto cleanup_failure;
  }

  struct sockaddr_in client_address;
  socklen_t client_len;
  int client_socket;

  while (1) {
    client_len = sizeof(struct sockaddr_in);
    client_socket =
        accept(server, (struct sockaddr *)&client_address, &client_len);
    if (client_socket < 0) {
      perror("accept");
      goto cleanup_failure;
    }

    // TODO: add server things here.

    close(client_socket);
  }
  close(server);
  return EXIT_SUCCESS;

cleanup_failure:
  close(server);
  return EXIT_FAILURE;
}
```

As mentioned earlier, we're only interested in parsing one request per connection, so let's block until we have a message in a buffer, then close the socket.

```c
int n_bytes;
char buffer[1024];

n_bytes = read(client_socket, buffer, sizeof(buffer) - 1);
if (n_bytes > 0) {
  buffer[n_bytes] = 0; // terminate with null

  // TODO: parse buffer and do cool things
}
```

Referring back to the request spec, we can split up the first line of the request into its constituent parts (method, path, HTTP version). In a more complete server, you would probably want to parse out the query parameters, headers, and request body to do something meaningful with it, but this is good enough for now.

To do the splitting, I'm going to rely heavily on `strtok`. It's worth mentioning that this modifies the source buffer to add `NULL`s, but that's perfectly fine for us.

```c
#include <string.h>

// <Method> <Path> <HTTP-Version>\r\n
char *method = strtok(buffer, " ");
char *path = strtok(NULL, " ");
char *http_version = strtok(NULL, "\r");

// The path technically consists of a path and a query string
// so let's split them up.
path = strtok(path, "?");
char *query = strtok(NULL, "");
```

Finally, let's generate a response containing these in the body and send it back to the client. Again, referring to the example above, we know we need a status line, a `Content-Type` header (without this, Chrome tries to download the response as a file), and the actual body itself.

```c
char response[1024];

sprintf(response,
        "%s 200 OK\r\n"
        "Content-Type: text/html\r\n"
        "\r\n"
        "You sent a <b>%s</b> request "
        "to <b>%s</b> with these params: <b>%s</b>",
        http_version, method, path, query);
write(client_socket, response, strlen(response));
```

And just like that, we've built the world's simplest, least compliant, and possibly worst HTTP server!

In a real server, you'd probably want to handle a ton of things that we're not doing here.

- Timeouts (a client can keep the socket open indefinitely)
- Request format validation (try sending a dummy string through `nc`)
- Routing (using `path` and friends)
- Parsing the request query string and body
- Parallelization (though maybe you could cheat this with GNU `parallel` and `SO_REUSEPORT`)

## Motivation

So why go through all of this? I found this super cool library called [Chafa](https://hpjansson.org/chafa/) which lets you show images in your terminal (through a mix of ASCII art and various terminal image protocols), and I wanted to use it to build a "now playing" widget for Spotify.

Spotify's API (like so many others) uses the OAuth flow if you want to get user data, and in this case, that's exactly what I need. One HTTP-server-rabbit-hole later, I finally have a [complete auth flow in my CLI](https://github.com/preyneyv/spotify-now-playing/blob/734edaf2dec78ad3a6980a04e72bdeeeb30eb5e1/src/spotify.c#L147-L242).

You can see the source code for my transient server implementation [here](https://github.com/preyneyv/spotify-now-playing/blob/734edaf2dec78ad3a6980a04e72bdeeeb30eb5e1/src/http-server.c). I'm still super new to C, so consider yourself warned.

## Resources

- [MDN HTTP Messages](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages)
- [HTTP/1.1 Specification (RFC 2616)](https://www.w3.org/Protocols/rfc2616/rfc2616.html)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/html/)

## Appendix

#### Final Server Code

```c
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

#define SERVER_PORT 1337

int main(void) {
  int server = socket(AF_INET, SOCK_STREAM, 0);

  int enable = 1;
  setsockopt(server, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));

  struct sockaddr_in address = {.sin_family = AF_INET,
                                .sin_addr = {.s_addr = htonl(INADDR_ANY)},
                                .sin_port = htons(SERVER_PORT)};
  if (bind(server, (struct sockaddr *)&address, sizeof(address)) == -1) {
    perror("bind");
    goto cleanup_failure;
  }

  if (listen(server, 5) == -1) {
    perror("listen");
    goto cleanup_failure;
  }

  struct sockaddr_in client_address;
  socklen_t client_len;
  int client_socket;
  int n_bytes;
  char buffer[1024];
  char response[1024];

  while (1) {
    client_len = sizeof(struct sockaddr_in);
    client_socket =
        accept(server, (struct sockaddr *)&client_address, &client_len);
    if (client_socket < 0) {
      perror("accept");
      goto cleanup_failure;
    }

    n_bytes = read(client_socket, buffer, sizeof(buffer) - 1);
    if (n_bytes > 0) {
      buffer[n_bytes] = 0; // terminate with null;

      // <Method> <Path> <HTTP-Version>\r\n
      char *method = strtok(buffer, " ");
      char *path = strtok(NULL, " ");
      char *http_version = strtok(NULL, "\r");

      // The path technically consists of a path and a query string
      // so let's split them up.
      path = strtok(path, "?");
      char *query = strtok(NULL, "");
      sprintf(response,
              "%s 200 OK\r\n"
              "Content-Type: text/html\r\n"
              "\r\n"
              "You sent a <b>%s</b> request "
              "to <b>%s</b> with these params: <b>%s</b>",
              http_version, method, path, query);
      write(client_socket, response, strlen(response));
    }

    close(client_socket);
  }
  close(server);
  return EXIT_SUCCESS;

cleanup_failure:
  close(server);
  return EXIT_FAILURE;
}
```
