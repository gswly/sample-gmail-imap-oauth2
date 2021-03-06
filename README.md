
# Howto: Gmail, Imap and Oauth2

Sample code of the Gmail-IMAP-Oauth2 authentication procedure, in Python and Go. Allows to perform automated operations on emails without toggling the "less secure apps" switch on the Google account page. Code is as simple as possible and is working as of 2019.

## Python

No dependencies are needed.

```python
from urllib.parse import urlencode
from urllib.request import urlopen, Request
import json
from imaplib import IMAP4_SSL

# create a Google app here https://console.developers.google.com
# then fill the following variables
GMAIL_CLIENT_ID = ""
GMAIL_CLIENT_SECRET = ""

# generate and print authorization link
url = "https://accounts.google.com/o/oauth2/auth?" + urlencode({
    "client_id": GMAIL_CLIENT_ID,
    "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
    "scope": "https://mail.google.com/ email",
    "response_type": "code",
})
print("visit\n%s\n" % url)

# read response code
code = input("paste reponse code: ")
print("")

# exchange code with access token
with urlopen(Request("https://accounts.google.com/o/oauth2/token", data=urlencode({
        "client_id": GMAIL_CLIENT_ID,
        "client_secret": GMAIL_CLIENT_SECRET,
        "code": code,
        "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
        "grant_type": "authorization_code",
    }).encode())) as res:
    eres = json.loads(res.read())

# request user email
with urlopen(Request("https://www.googleapis.com/oauth2/v2/userinfo", headers={
        "Authorization": "Bearer %s" % eres["access_token"],
    })) as res:
    ures = json.loads(res.read())

# connect to imap
imap = IMAP4_SSL("imap.gmail.com", 993)

# authenticate the gmail way
imap.authenticate("XOAUTH2", lambda x:
    "user=%s\1auth=Bearer %s\1\1" % (ures["email"], eres["access_token"]))

# the following is just an example that shows available folders
# you can use any function provided by imaplib
# https://docs.python.org/3/library/imaplib.html

print("available folders:")
res,data = imap.list('""', "*")
for mbox in data:
    print(mbox.decode())

print("")
```

## Go

First install the go-imap package:
```
go get github.com/emersion/go-imap/...
```

```go
package main

import (
    "os"
    "fmt"
    "bufio"
    "net/url"
    "net/http"
    "encoding/json"
    ilib "github.com/emersion/go-imap"
    iclient "github.com/emersion/go-imap/client"
    "github.com/emersion/go-sasl"
)

// create a Google app here https://console.developers.google.com
// then fill the following variables
const (
    GMAIL_CLIENT_ID = ""
    GMAIL_CLIENT_SECRET = ""
)

type ExchangeRes struct {
    AccessToken string `json:"access_token"`
}

type UserinfoRes struct {
    Email string `json:"email"`
}

func main() {
    // generate and print authorization link
    ourl := "https://accounts.google.com/o/oauth2/auth?"
    v := url.Values{
        "client_id": { GMAIL_CLIENT_ID },
        "redirect_uri": { "urn:ietf:wg:oauth:2.0:oob" },
        "scope": { "https://mail.google.com/ email" },
        "response_type": { "code" },
    }
    ourl += v.Encode()
    fmt.Printf("visit\n%s\n\n", ourl)

    // read response code
    fmt.Println("paste reponse code: ")
    reader := bufio.NewReader(os.Stdin)
    code,_ := reader.ReadString('\n')
    fmt.Println("")

    // exchange code with access token
    res,err := http.PostForm("https://accounts.google.com/o/oauth2/token", url.Values{
        "client_id": { GMAIL_CLIENT_ID },
        "client_secret": { GMAIL_CLIENT_SECRET },
        "code": { code },
        "redirect_uri": { "urn:ietf:wg:oauth:2.0:oob" },
        "grant_type": { "authorization_code" },
    })
    if err != nil {
        panic(err)
    }
    defer res.Body.Close()
    var eres ExchangeRes
    err = json.NewDecoder(res.Body).Decode(&eres)
    if err != nil {
        panic(err)
    }

    // request user email
    req,err := http.NewRequest("GET", "https://www.googleapis.com/oauth2/v2/userinfo", nil)
    if err != nil {
        panic(err)
    }
    req.Header.Add("Authorization", "Bearer " + eres.AccessToken)
    res,err = http.DefaultClient.Do(req)
    if err != nil {
        panic(err)
    }
    var ures UserinfoRes
    err = json.NewDecoder(res.Body).Decode(&ures)
    if err != nil {
        panic(err)
    }

    // connect to imap server
    imap,err := iclient.DialTLS("imap.gmail.com:993", nil)
    if err != nil {
        panic(err)
    }
    defer imap.Logout()

    // authenticate the gmail way
    err = imap.Authenticate(sasl.NewXoauth2Client(ures.Email, eres.AccessToken))
    if err != nil {
        panic(err)
    }

    // the following is just an example that shows available folders
    // more examples are available here
    // https://godoc.org/github.com/emersion/go-imap/client

    fmt.Println("available folders:")
    mailboxes := make(chan *ilib.MailboxInfo)
    done := make(chan error, 1)
    go func () {
        done <- imap.List("", "*", mailboxes)
    }()
    for m := range mailboxes {
        fmt.Println("* " + m.Name)
    }
    if err := <-done; err != nil {
        panic(err)
    }
}
```
