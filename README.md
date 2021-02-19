[<img src=".github/images/logo.png" align="right" width="256px"/>](https://devforum.okta.com/)
[![GitHub Workflow Status](https://github.com/okta/okta-idx-golang/workflows/CI/badge.svg)](https://github.com/okta/okta-idx-golang/actions)
[![Go Report Card](https://goreportcard.com/badge/github.com/okta/okta-idx-golang?style=flat-square)](https://goreportcard.com/report/github.com/okta/okta-idx-golang)
![Go Version](https://img.shields.io/badge/go%20version-%3E=1.14-61CFDD.svg?style=flat-square)
[![PkgGoDev](https://pkg.go.dev/badge/mod/github.com/okta/okta-idx-golang)](https://pkg.go.dev/mod/github.com/okta/okta-idx-golang)

# Okta IDX - Golang

This repository contains the Okta IDX SDK for Golang. This SDK can be used in your server-side code to assist in authenticating users against the Okta IDX.


> :grey_exclamation: The use of this SDK requires the usage of the Okta Identity Engine. This functionality is in general availability but is being gradually rolled out to customers. If you want to request to gain access to the Okta Identity Engine, please reach out to your account manager. If you do not have an account manager, please reach out to oie@okta.com for more information.
## Release status

This library uses semantic versioning and follows Okta's [Library Version Policy][okta-library-versioning].

| Version | Status                             |
| ------- | ---------------------------------- |
| 0.x     | :warning: In Development           |

The latest release can always be found on the [releases page][github-releases].


## Need help?

If you run into problems using the SDK, you can

 - Ask questions on the [Okta Developer Forums][devforum]
 - Post [issues on GitHub][github-issues] (for code errors)


## Getting started

### Prerequisites
You will need:
 - An Okta account, called an organization. (Sign up for a free [developer organization][developer-edition-signup] if you need one)
 - Access to the Okta Identity Engine feature. Currently, an early access feature. Contact [support@okta.com][support-email] for more information.

## Usage Guide
These examples will help you understand how to use this library.

Once you initialize a `Client`, you can call methods to make requests to the Okta IDX API.

### Create the Client
```go
client, err := NewClient(
    WithClientID("{YOUR_CLIENT_ID}"),
    WithClientSecret("{YOUR_CLIENT_SECRET}"),   // Required for confidential clients.
    WithIssuer("{YOUR_ISSUER}"),                // e.g. https://foo.okta.com/oauth2/default, https://foo.okta.com/oauth2/ausar5vgt5TSDsfcJ0h7
    WithScopes([]string{"openid", "profile"}),  // Must include at least `openid`. Include `profile` if you want to do token exchange
    WithRedirectURI("{YOUR_REDIRECT_URI}"),     // Must match the redirect uri in client app settings/console
)
if err != nil {
    fmt.Errorf("could not create a new IDX Client", err)
}
```

### Get Context
The Context is where we store the interactionHandle, state, and codeVerifier for your application for you.
```go
idxContext, err := IDXClient.Interact(context.TODO(), nil)
if err != nil {
    fmt.Errorf("retriving an interaction handle failed", err)
}
```

### Using Interaction Handle for Introspect
```go
introspectResponse, err := IDXClient.Introspect(context.TODO(), interactionHandle)
if err != nil {
    fmt.Errorf("could not introspect IDX", err)
}
```

#### Get New Tokens (access_token/id_token/refresh_token)
In this example, the sign-on policy has no authenticators required.
> Note: Steps to identify the user might change based on the Org configuration.

```go
var response *Response

client, err := NewClient(
    WithClientID("{CLIENT_ID}"),
    WithClientSecret("{CLIENT_SECRET}"),        // Required for confidential clients.
    WithIssuer("{ISSUER}"),                     // e.g. https://foo.okta.com/oauth2/default, https://foo.okta.com/oauth2/busar5vgt5TSDsfcJ0h7
    WithScopes([]string{"openid", "profile"}),  // Must include at least `openid`. Include `profile` if you want to do token exchange
    WithRedirectURI("{REDIRECT_URI}"),          // Must match the redirect uri in client app settings/console
)
if err != nil {
    panic(err)
}

idxContext, err := client.Interact(context.TODO(), nil)
if err != nil {
    panic(err)
}

response, err = client.Introspect(context.TODO(), idxContext)
if err != nil {
    panic(err)
}

for !response.LoginSuccess() {
    for _, remediationOption := range response.Remediation.RemediationOptions {

        switch remediationOption.Name {
        case "identify":
            identify := []byte(`{
                    "identifier": "foo@example.com",
                    "rememberMe": false
                }`)

            response, err = remediationOption.Proceed(context.TODO(), identify)
            if err != nil {
                panic(err)
            }

        case "challenge-authenticator":
            credentials := []byte(`{
                    "credentials": {
                    "passcode": "Abcd1234"
                    }
                }`)

            response, err = remediationOption.Proceed(context.TODO(), credentials)

            if err != nil {
                panic(err)
            }

        default:
            fmt.Printf("%+v\n", response.Remediation)
            panic("could not handle")
        }

    }
}

// These properties are based on the `successWithInteractionCode` object, and the properties that you are required to fill out
exchangeForm := []byte(`{
    "client_secret": "` + client.ClientSecret() + `",
    "code_verifier": "` + idxContext.CodeVerifier() + `" // We generate your code_verfier for you and store it in the Context struct. You can gain access to it through the method `CodeVerifier()` which will return a string
}`)
tokens, err := response.SuccessResponse.ExchangeCode(context.Background(), exchangeForm)
if err != nil {
    panic(err)
}

fmt.Printf("Tokens: %+v\n", tokens)
fmt.Printf("Access Token: %+s\n", tokens.AccessToken)
fmt.Printf("ID Token: %+s\n", tokens.IDToken)
```

#### Enroll + Login using password + email authenticator

```go
   var response *Response

    client, err := NewClient(
        WithClientID("{CLIENT_ID}"),
        WithClientSecret("{CLIENT_SECRET}"),        // Required for confidential clients.
        WithIssuer("{ISSUER}"),                     // e.g. https://foo.okta.com/oauth2/default, https://foo.okta.com/oauth2/busar5vgt5TSDsfcJ0h7
        WithScopes([]string{"openid", "profile"}),  // Must include at least `openid`. Include `profile` if you want to do token exchange
        WithRedirectURI("{REDIRECT_URI}"),          // Must match the redirect uri in client app settings/console
    )
	if err != nil {
		panic(err)
	}

	idxContext, err := client.Interact(context.TODO(), nil)
	if err != nil {
		panic(err)
	}

	response, err = client.Introspect(context.TODO(), idxContext)
	if err != nil {
		panic(err)
	}

	// First remediation will be "identify"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "identify" {
			identify := []byte(`{
                    "identifier": "foo@example.com",
                    "rememberMe": false
                }`)

			response, err = remediationOption.Proceed(context.TODO(), identify)
			if err != nil {
				panic(err)
			}
		} else {
			panic("we expected an `identify` option, but did not see one.")
		}
	}

	// choosing password as an authenticator
	for _, remediationOption := range response.Remediation.RemediationOptions {
		var id string

		for _, options := range remediationOption.FormValues[0].Options {
			if options.Label == "Password" {
				id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
			}
		}

		authenticator := []byte(`{
			"authenticator": {
			"id": "` + id + `"
			}
		}`)

		response, err = remediationOption.Proceed(context.TODO(), authenticator)
		if err != nil {
			panic(err)
		}
	}

	// challenge-authenticator
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "challenge-authenticator" {
			credentials := []byte(`{
				"credentials": {
					"passcode": "Abcd1234"
				}
			}`)
			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
		}
	}

    // choosing email as an authenticator
	for _, remediationOption := range response.Remediation.RemediationOptions {
		var id string
		for _, options := range remediationOption.FormValues[0].Options {
			fmt.Printf("%+v\n", options)

			if options.Label == "Email" {
				id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
			}
		}

		authenticator := []byte(`{
			"authenticator": {
			"id": "` + id + `"
			}
		}`)

		response, err = remediationOption.Proceed(context.TODO(), authenticator)
		if err != nil {
			panic(err)
		}
	}

	// Next remediation will be "challenge-authenticator"
	// You should receive an email with the passcode and then enter it here
	for _, remediationOption := range response.Remediation.RemediationOptions {
		fmt.Println("Enter Your Email Passcode: ")

		// var then variable name then variable type
		var passcode string

		// Taking input from user
		fmt.Scanln(&passcode)

		if remediationOption.Name == "challenge-authenticator" {
			credentials := []byte(`{
				"credentials": {
					"passcode": "` + passcode + `"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
		} else {
			panic("we expected an `challenge-authenticator` option, but did not see one.")
		}
	}

	if response.LoginSuccess() {
		fmt.Println("Successful login!")
		// These properties are based on the `successWithInteractionCode` object, and the properties that you are required to fill out
		exchangeForm := []byte(`{
			"client_secret": "` + client.ClientSecret() + `",
			"code_verifier": "` + idxContext.CodeVerifier() + `" // We generate your code_verfier for you and store it in the Context struct. You can gain access to it through the method `CodeVerifier()` which will return a string
		}`)
		tokens, err := response.SuccessResponse.ExchangeCode(context.Background(), exchangeForm)
		if err != nil {
		    panic(err)
		}

		fmt.Printf("%+v\n", tokens)
		fmt.Printf("%+s\n", tokens.AccessToken)
		fmt.Printf("%+s\n", tokens.IDToken)
        fmt.Printf("%+s\n", tokens.RefreshToken)
	} else {
        panic("we expected successful login")
    }
```

#### Enroll + Login using password + security question authenticator
```go
    // here goes the part same as for 'Enroll + Login using password + email authenticator'
    // up to  'choosing email as an authenticator'

    // choosing security question as an authenticator
    for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "select-authenticator-enroll" {
			var id string
			for _, options := range remediationOption.FormValues[0].Options {
				if (options.Label == "Security Question") {
					id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				}
			}

			authenticator := []byte(`{
				"authenticator": {
					"id": "` + id + `"
				},
				"method_type": "security_question"
			}`)

			response, err = remediationOption.Proceed(context.TODO(), authenticator)
			if err != nil {
				panic(err)
			}

		} else {
			panic("we expected an `select-authenticator-enroll` option, but did not see one.")
		}
	}

	// Next remediation will be "enroll-authenticator"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "enroll-authenticator" {
			var question string

			for _, options := range remediationOption.FormValues[0].Options {
				if (options.Label == "Choose a security question") {
					question = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Name
				}
			}
			answer := []byte(`{
				"credentials": {
					"questionKey": "disliked_food",
					"answer": "salad"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), answer)
			if err != nil {
				panic(err)
			}

		}
	}

	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "skip" {
			skip := []byte(`{}`)

			response, err = remediationOption.Proceed(context.TODO(), skip)
			if err != nil {
				panic(err)
			}
		}
	}

	if response.LoginSuccess() {
        fmt.Println("Successful login!")
        // get the token
    }
```

#### Enroll + Login using password + SMS authenticator

```go
    // here goes the part same as for 'Enroll + Login using password + email authenticator'
    // up to  'choosing email as an authenticator'

    // choosing phone as an authenticator
    for _, remediationOption := range response.Remediation.RemediationOptions {
		var id string
		var enrollmentId string
		for _, options := range remediationOption.FormValues[0].Options {
			fmt.Printf("%+v\n", options)

			if (options.Label == "Phone") {
				id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				enrollmentId = options.Value.(idx.FormOptionsValueObject).Form.Value[2].Value
			}
		}

		authenticator := []byte(`{
			"authenticator": {
				"id": "` + id + `",
				"enrollmentId": "` + enrollmentId + `",
				"methodType": "sms"
			}
		}`)

		response, err = remediationOption.Proceed(context.TODO(), authenticator)
		if err != nil {
			panic(err)
		}
	}

	// Next remediation will be "challenge-authenticator"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		fmt.Println("Enter Your SMS Passcode: ")

		// var then variable name then variable type
		var passcode string

		fmt.Printf("%+v\n", remediationOption.Form())
	    if remediationOption.Name == "challenge-authenticator" {
			// Taking input from user
			fmt.Scanln(&passcode)

			credentials := []byte(`{
				"credentials": {
					"passcode": "` + passcode + `"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
	    }
	}

	if (response.LoginSuccess()) {
		fmt.Println("Successful login!")
        // get the token
	}

```

#### Password reset flow

```go
    // here goes the part same as in Enroll + Login using password + email authenticator'
    // up to  'choosing password as an authenticator'

	// this step is different from others, because we want to reset the password
	if response.CurrentAuthenticatorEnrollment != nil {
		response, err = response.CurrentAuthenticatorEnrollment.Value.Recover.Proceed(context.TODO(), nil)
		if err != nil {
			panic(err)
		}
	} else {
		panic("we expected an `CurrentAuthenticatorEnrollment` to reset the password, but did not see one.")
	}

	// send me reset code on email
	for _, remediationOption := range response.Remediation.RemediationOptions {
		var id string
		for _, options := range remediationOption.FormValues[0].Options {
			if options.Label == "Email" {
				id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				break
			}
		}
		if id == "" {
			continue
		}
		authenticator := []byte(`{
			"authenticator": {
			"id": "` + id + `"
			}
		}`)
		response, err = remediationOption.Proceed(context.TODO(), authenticator)
		if err != nil {
			panic(err)
		}
		break
	}

	reader := bufio.NewReader(os.Stdin)
	fmt.Print("Enter the code from email: ")
	text, _ := reader.ReadString('\n')

	// Next remediation will be "challenge-authenticator". In this case entering code from email
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "challenge-authenticator" {
			credentials := []byte(fmt.Sprintf(`{
				"credentials": {
					"passcode": "%s"
				}
			}`, strings.TrimSpace(text)))
			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
			break
		}
	}

	// Next remediation will be "challenge-authenticator". In this case answering the question
	for _, remediationOption := range response.Remediation.RemediationOptions {
		fmt.Printf("%+v\n", remediationOption.Form())
		if remediationOption.Name == "challenge-authenticator" {
			fmt.Print("Enter the food you dislike: ")
			text, _ = reader.ReadString('\n')
			credentials := []byte(`{
				"credentials": {
					"answer": "` + strings.TrimSpace(text) + `"
				}
			}`)
			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
		}
	}

	// Next remediation will be "reset-authenticator" as we want to enter new passcode
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "reset-authenticator" {
			fmt.Print("Enter new password: ")
			text, _ = reader.ReadString('\n')
			credentials := []byte(fmt.Sprintf(`{
				"credentials": {
					"passcode": "%s"
				}
			}`, strings.TrimSpace(text)))
			response, err = remediationOption.Proceed(context.TODO(), credentials)
			if err != nil {
				panic(err)
			}
			break
		}
	}

	// if new password was set, we should get successful login!
	if response.LoginSuccess() {
		fmt.Println("Successful login!")
        // get the token
		exchangeForm := []byte(`{
			"client_secret": "` + client.ClientSecret() + `",
			"code_verifier": "` + idxContext.CodeVerifier() + `" // We generate your code_verfier for you and store it in the Context struct. You can gain access to it through the method `CodeVerifier()` which will return a string
		}`)
		tokens, err := response.SuccessResponse.ExchangeCode(context.Background(), exchangeForm)
		if err != nil {
		    panic(err)
		}
		fmt.Printf("%+v\n", tokens)
		fmt.Printf("%+s\n", tokens.AccessToken)
        fmt.Printf("%+s\n", tokens.IDToken)
	}
```

#### Cancel the OIE Transaction and Start a New One
In this example the Org is configured to require email as a second authenticator. After answering the password challenge, a cancel request is sent right before answering the email challenge.

```go
var response *Response

client, err := NewClient(
    WithClientID("{CLIENT_ID}"),
    WithClientSecret("{CLIENT_SECRET}"),       // Required for confidential clients.
    WithIssuer("{ISSUER}"),                    // e.g. https://foo.okta.com/oauth2/default, https://foo.okta.com/oauth2/ausar5vgt5TSDsfcJ0h7
    WithScopes([]string{"openid", "profile"}), // Must include at least `openid`. Include `profile` if you want to do token exchange
    WithRedirectURI("{REDIRECT_URI}"),         // Must match the redirect uri in client app settings/console
)
if err != nil {
    panic(err)
}

_, err = client.Interact(context.TODO())
if err != nil {
    panic(err)
}

idxContext, err := client.Interact(context.TODO(), nil)
if err != nil {
    panic(err)
}

response, err = client.Introspect(context.TODO(), idxContext)
if err != nil {
    panic(err)
}

for _, remediationOption := range response.Remediation.RemediationOptions {

    if remediationOption.Name == "identify" {
        identify := []byte(`{
                "identifier": "foo@example.com",
                "rememberMe": false
            }`)

        response, err = remediationOption.Proceed(context.TODO(), identify)
        if err != nil {
            panic(err)
        }
    } else {
        panic("we expected an `identify` option, but did not see one.")
    }
}

for _, remediationOption := range response.Remediation.RemediationOptions {

    if remediationOption.Name == "challenge-authenticator" {
        credentials := []byte(`{
                "credentials": {
                "passcode": "Abcd1234"
                }
            }`)

        response, err = remediationOption.Proceed(context.TODO(), credentials)

        if err != nil {
            panic(err)
        }
    } else {
        panic("we expected an `identify` option, but did not see one.")
    }
}

response, err := response.Cancel(context.TODO())
if err != nil {
    panic(err)
}

// From now on, you can use response to continue with a new flow. You will notice here that you have a new `stateHandle` which signals a new flow. Your `interaction_handle` will remain the same.
```

#### Self Service Registration

```go
	var response *Response

    client, err := NewClient(
        WithClientID("{CLIENT_ID}"),
        WithClientSecret("{CLIENT_SECRET}"),        // Required for confidential clients.
        WithIssuer("{ISSUER}"),                     // e.g. https://foo.okta.com/oauth2/default, https://foo.okta.com/oauth2/busar5vgt5TSDsfcJ0h7
        WithScopes([]string{"openid", "profile"}),  // Must include at least `openid`. Include `profile` if you want to do token exchange
        WithRedirectURI("{REDIRECT_URI}"),          // Must match the redirect uri in client app settings/console
    )
	if err != nil {
		panic(err)
	}

	idxContext, err := client.Interact(context.TODO(), nil)
	if err != nil {
		panic(err)
	}

	response, err = client.Introspect(context.TODO(), idxContext)
	if err != nil {
		panic(err)
	}

	// First remediation will be "identify"
	for _, remediationOption := range response.Remediation.RemediationOptions {
	    if remediationOption.Name == "select-enroll-profile" {
	        enroll := []byte(`{}`)

	        response, err = remediationOption.Proceed(context.TODO(), enroll)
	        if err != nil {
	            panic(err)
	        }
	    }
	}

	// Next remediation will be "enroll-profile"
	for _, remediationOption := range response.Remediation.RemediationOptions {

		fmt.Printf("%+v\n", remediationOption.Form())
		if remediationOption.Name == "enroll-profile" {
			userProfile := []byte(`{
				"userProfile": {
					"lastName": "User",
					"firstName": "Test",
					"email": "john.doe@acme.com"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), userProfile)
			if err != nil {
				panic(err)
			}
		}
	}

	// Next remediation will be "select-authenticator-enroll"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "select-authenticator-enroll" {
			var id string

			for _, options := range remediationOption.FormValues[0].Options {
				fmt.Printf("%+v\n", options)

				if (options.Label == "Security Question") {
					id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				}
			}

			authenticator := []byte(`{
				"authenticator": {
					"id": "` + id + `"
				},
				"method_type": "security_question"
			}`)

			response, err = remediationOption.Proceed(context.TODO(), authenticator)
			if err != nil {
				panic(err)
			}

		} else {
			panic("we expected an `select-authenticator-enroll` option, but did not see one.")
		}
	}

	// Next remediation will be "enroll-authenticator"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "enroll-authenticator" {
			var question string

			for _, options := range remediationOption.FormValues[0].Options {
				fmt.Printf("%+v\n", options)

				if (options.Label == "Choose a security question") {
					question = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Name
				}
			}

			fmt.Printf(question)

			answer := []byte(`{
				"credentials": {
					"questionKey": "disliked_food",
					"answer": "Mushrooms"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), answer)
			if err != nil {
				panic(err)
			}

		}
	}

	// Next remediation will be "select-authenticator-enroll"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "select-authenticator-enroll" {
			var id string

			for _, options := range remediationOption.FormValues[0].Options {
				fmt.Printf("%+v\n", options)

				if (options.Label == "Password") {
					id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				}
			}

			authenticator := []byte(`{
				"authenticator": {
					"id": "` + id + `"
				},
				"method_type": "password"
			}`)

			response, err = remediationOption.Proceed(context.TODO(), authenticator)
			if err != nil {
				panic(err)
			}

		} else {
			panic("we expected an `select-authenticator-enroll` option, but did not see one.")
		}
	}

	// Next remediation will be "enroll-authenticator"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "enroll-authenticator" {
			var question string

			for _, options := range remediationOption.FormValues[0].Options {
				fmt.Printf("%+v\n", options)

				if (options.Label == "Choose a security question") {
					question = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Name
				}
			}

			fmt.Printf(question)

			answer := []byte(`{
				"credentials": {
					"passcode": "Password!"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), answer)
			if err != nil {
				panic(err)
			}

		}
	}

	// Next remediation will be "select-authenticator-enroll"
	for _, remediationOption := range response.Remediation.RemediationOptions {
		if remediationOption.Name == "select-authenticator-enroll" {
			var id string

			for _, options := range remediationOption.FormValues[0].Options {
				fmt.Printf("%+v\n", options)

				if (options.Label == "Email") {
					id = options.Value.(idx.FormOptionsValueObject).Form.Value[0].Value
				}
			}

			authenticator := []byte(`{
				"authenticator": {
					"id": "` + id + `"
				},
				"method_type": "email"
			}`)

			response, err = remediationOption.Proceed(context.TODO(), authenticator)
			if err != nil {
				panic(err)
			}
		}
	}

	// Next remediation will be "challenge-authenticator"
	for _, remediationOption := range response.Remediation.RemediationOptions {

		fmt.Printf("%+v\n", remediationOption.Form())
	    if remediationOption.Name == "enroll-authenticator" {
			fmt.Println("Enter Your Email Passcode: ")

			var passcode string

			// Taking input from user
			fmt.Scanln(&passcode)

			credentials := []byte(`{
				"credentials": {
					"passcode": "` + passcode + `"
				}
			}`)

			response, err = remediationOption.Proceed(context.TODO(), credentials)

			if err != nil {
				panic(err)
			}
	    }
	}


	for _, remediationOption := range response.Remediation.RemediationOptions {

		fmt.Printf("%+v\n", remediationOption.Form())
	    if remediationOption.Name == "skip" {
			skip := []byte(`{}`)

			response, err = remediationOption.Proceed(context.TODO(), skip)
	        if err != nil {
	            panic(err)
	        }
	    }
	}

	fmt.Println(string(response.Raw()))

	if (response.LoginSuccess()) {
		fmt.Println("Successful login!")
		exchangeCodeForTokens(response, client)
	}
```

### Check Remediation Options
```go
// check remediation options to continue the flow
options := idxResponse.Remediation.RemediationOptions
option := options[:0]
formValues := option.Form()

```

### Cancel Flow
You can cancel the current flow at any time. This will invalidate the current `stateHandle` and return a new remediation response with a new `stateHandle`.
```go
idxResponse, err := response.Cancel(context.TODO())
```

### Get Raw Response
At times, you may need to access the full response. This can be done with any IDX response:
```go
raw := response.Raw()
```

### Determine When Login is Successful
At any point during the login, you may be finished with remedidation. For this, we provide a `LoginSuccess()` method you can check which will return a boolean.
```go
isLoginSuccess := response.LoginSuccess()
```

## Configuration Reference
This library looks for the configuration in the following sources:

1. An okta.yaml file in a .okta folder in the current user's home directory (~/.okta/okta.yaml or %userprofile%\.okta\okta.yaml)
2. An okta.yaml file in a .okta folder in the application or project's root directory
3. Environment variables
4. Configuration explicitly passed to the constructor (see the example in [Getting started](#getting-started))

Higher numbers win. In other words, configuration passed via the constructor will override configuration found in environment variables, which will override configuration in okta.yaml (if any), and so on.

### Config Properties
| Yaml Path             | Environment Key       | Description                                                                                                          |
|-----------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------|
| okta.idx.issuer       | OKTA_IDX_ISSUER       | The issuer of the authorization server you want to use for authentication.                                           |
| okta.idx.clientId     | OKTA_IDX_CLIENTID     | The client ID of the Okta Application.                                                                               |
| okta.idx.clientSecret | OKTA_IDX_CLIENTSECRET | The client secret of the Okta Application. Required with confidential clients                                        |
| okta.idx.scopes       | OKTA_IDX_SCOPES       | The scopes requested for the access token.                                                                           |
| okta.idx.redirectUri  | OKTA_IDX_REDIRECTURI  | For most cases, this will not be used, but is still required to supply. You can put any configured redirectUri here. |

#### Yaml Configuration
The configuration would be expressed in our okta.yaml configuration for SDKs as follows:

```yaml
okta:
  idx:
    issuer: {issuerUrl}
    clientId: {clientId}
    clientSecret: {clientSecret}
    scopes:
    - {scope1}
    - {scope2}
    redirectUri: {configuredRedirectUri}
```

#### Environment Configuration
The configuration would be expressed in environment variables for SDKs as follows:
```env
OKTA_IDX_ISSUER
OKTA_IDX_CLIENTID
OKTA_IDX_CLIENTSECRET
OKTA_IDX_SCOPES
OKTA_IDX_REDIRECTURI
```


[okta-library-versioning]: https://developer.okta.com/code/library-versions/
[github-issues]: https://github.com/okta/okta-idx-golang/issues
[developer-edition-signup]: https://developer.okta.com/signup
[support-email]: mailto://support@okta.com