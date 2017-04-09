Two-factor authentication (2FA) simple refers to the process of securing something with 2 things.  It's also referred to as "multi-factor authentication" (MFA) but obviously in this case we're only dealing with two things.  It boils down to this:

1. Something you have
2. Something you know 

In a system without 2FA, someone could gain access to your private stash of kitten videos by getting hold of your password.  Be that through guessing [see here](https://xkcd.com/936/), phishing, or the more traditional rubber-hose approach.

In a system secured by 2FA, if someone got hold of your password, they'd still need to *have* your access token.  There are two major types of access token, both based on [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) but with different increment schedules.

###### Algorithms

**HOTP** (RFC 4226) event-based

A counter-based algorithm, where the moving factor is an incrementing counter (updates after every *successful* use).  If compromised, tokens can be used by until the counter shifts again; plenty of time for someone to bruteforce your password.

**TOTP** (RFC 6238) time-based

A time-based algorithm, where the moving factor is time.  If compromised, tokens are only valid for a few seconds, every time the counter shifts (typically 30 secons), our bruteforcer has to start again.

**Summary**

When using 2FA, the following are true:

* If just your password is compromised, your data is safe
* If just your access token is compromised, your data is safe
* If your password *and* access token are compromised, your data will be safe again when the OTP counter increments

###### Workflow

Here's a working example of 2FA that protects users of the new and fantastically popular new site kittenfrivolity.gov.org.

The following package makes all of this possible:

[github.com/pquerna/otp](https://github.com/pquerna/otp)

1.  User registers for 2FA and a random secret is generated, base32 encoded and stored against their account.

``` go
func register(email string, issuer string) (key *otp.Key, err error) {
	if key, err = totp.Generate(totp.GenerateOpts{
		Issuer:      issuer,
		AccountName: email,
		SecretSize:  20,
	}); err != nil {
		return
	}

	// TODO: persist key.Secret() to DB

	return
}
```

Let's assume the output of the above function is the following is `JBSWY3DPEHPK3PXP` and has been persisted for the user.

3.  A URL is generated and QR encoded to an email the user can scan with with phone:

``` go
func generateQR(key *otp.Key, writer io.Writer) (err error) {
	var img image.Image
	if img, err = key.Image(200, 200); err != nil {
		return
	}

	return png.Encode(writer, img)
}
```

This URL includes the user's email address, the issuer of the token (kittenfrivolity.gov.org) and their secret.  Here's the decoded QR contents:

  otpauth://totp/kittenfrivolity:rob@kittenfrivolity.gov.org?secret=JBSWY3DPEHPK3PXP&issuer=kittenfrivolity

5.  The user scans the QR code using their OTP authentication app.

This gives the authentication app all the information it needs to generate tokens and help the user identify which token they'll need to use to access all those kitten videos.

6.  Access token generation begins

The user enters the code in their authentication app and submits it to the server.  The server generates its own token (using the stored secret and the two are compared):

``` go
func validate(password string) (ok bool, err error) {
	var secret string
    // TODO: read secret from DB

	totp.Validate(passcode, secret)
}
```
# I don't like the above code, needs improving