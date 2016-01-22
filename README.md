# About jwt [![Build Status](https://travis-ci.org/knq/jwt.svg)](https://travis-ci.org/knq/jwt) [![Coverage Status](https://coveralls.io/repos/knq/jwt/badge.svg?branch=master&service=github)](https://coveralls.io/github/knq/jwt?branch=master) #

A [Golang](https://golang.org/project) package that provides a simple and
secure way to encode and decode [JWT](https://jwt.io/) tokens.

## Installation ##

Install the package via the following:

    go get -u github.com/knq/jwt

## Usage ##

Please see [the GoDoc API page](http://godoc.org/github.com/knq/jwt) for a
full API listing.

The jwt package can be used similarly to the following:

```go
// example/example.go
package main

//go:generate openssl genrsa -out rsa-private.pem 2048
//go:generate openssl rsa -in rsa-private.pem -outform PEM -pubout -out rsa-public.pem

import (
    "fmt"
    "log"
    "reflect"
    "time"

    "github.com/knq/jwt"
)

func main() {
    // create PS384 with private and public key
    // in addition, there are the other standard JWT encryption implementations:
    // HMAC:         HS256, HS384, HS512
    // RSA-PKCS1v15: RS256, RS384, RS512
    // ECC:          ES256, ES384, ES512
    // RSA-SSA-PSS:  PS256, PS384, PS512
    ps384 := jwt.PS384.New(jwt.PEM{"rsa-private.pem", "rsa-public.pem"})

    // calculate an expiration time we have to clear the nanoseconds, otherwise
    // DeepEqual won't be true later, as the precision in the JWT standard for
    // time is only seconds.
    t := time.Now().Add(14 * 24 * time.Hour)
    t = t.Add(time.Duration(-t.Nanosecond()))

    // wrap expiration time as jwt.ClaimsTime
    expr := jwt.ClaimsTime(t)

    // create claims using provided jwt.Claims
    c0 := jwt.Claims{
        Issuer:     "user@example.com",
        Audience:   "client@example.com",
        Expiration: &expr,
    }
    fmt.Printf("Claims: %+v\n\n", c0)

    // encode token
    buf, err := ps384.Encode(&c0)
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Printf("Encoded token:\n\n%s\n\n", string(buf))

    // decode the generated token and verify
    c1 := jwt.Claims{}
    err = ps384.Decode(buf, &c1)
    if err != nil {
        // if the signature was bad, the err would not be nil
        log.Fatalln(err)
    }
    if reflect.DeepEqual(c0, c1) {
        fmt.Printf("Claims Match! Decoded claims: %+v\n\n", c1)
    }

    fmt.Println("----------------------------------------------")

    // use custom claims
    c3 := map[string]interface{}{
        "aud": "my audience",
        "http://example/api/write": true,
    }
    fmt.Printf("My Custom Claims: %+v\n\n", c3)

    buf, err = ps384.Encode(&c3)
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Printf("Encoded token with custom claims:\n\n%s\n\n", string(buf))

    // decode custom claims
    c4 := myClaims{}
    err = ps384.Decode(buf, &c4)
    if err != nil {
        log.Fatalln(err)
    }
    if c4.Audience == "my audience" {
        fmt.Printf("Decoded custom claims: %+v\n\n", c1)
    }
    if c4.WriteScope {
        fmt.Println("myClaims custom claims has write scope!")
    }
}

// or use any type that the standard encoding/json library recognizes as a
// payload (you can also "extend" jwt.Claims in this fashion):
type myClaims struct {
    jwt.Claims
    WriteScope bool `json:"http://example/api/write"`
}
```