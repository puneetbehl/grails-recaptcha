[![Stories in Ready](https://badge.waffle.io/iamthechad/grails-recaptcha.png?label=ready&title=Ready)](https://waffle.io/iamthechad/grails-recaptcha)
# Version 1.0 Notice

Beginning with version 1.0 of this plugin, only the new "checkbox" captcha is supported. Please use version 0.7.0 if you require the legacy functionality.

**Note:** This plugin currently only supports the "traditional" captcha use case of automatic rendering. Explicit rendering will be available soon. (See [the ReCaptcha docs](https://developers.google.com/recaptcha/docs/display) for more information about automatic vs. explicit.

# Introduction

This plugin is designed to make using the ReCaptcha and Mailhide services within Grails easy. In order to use this plugin, you must have a ReCaptcha account, available from [http://www.google.com/recaptcha](http://www.google.com/recaptcha).

# Installation

Add the following to your `grails-app/conf/BuildConfig.groovy`

    …
    plugins {
        …
        compile ':recaptcha:1.1.0'
        …
    }
    
Upon installation, run `grails recaptcha-quickstart` to create the skeleton configuration. The quickstart has two targets: `integrated` or `standalone`, depending on where you'd like the configuration to live.

## Integrated Configuration
Integrated configuration adds the following to the end of your `Config.groovy` file:

    // Added by the Recaptcha plugin:
    recaptcha {
        // These keys are generated by the ReCaptcha service
        publicKey = ""
        privateKey = ""

        // Include the noscript tags in the generated captcha
        includeNoScript = true

        // Set to false to disable the display of captcha
        enabled = true
    }

    mailhide {
        // Generated by the Mailhide service
        publicKey = ""
        privateKey = ""
    }
    
This configuration can be modified to mimic the standalone if there is a need for different behavior depending on the current environment.

## Standalone Configuration
Standalone configuration creates a file called  `RecaptchaConfig.groovy`  in  `grails-app/conf` with the following content:

	recaptcha {
	    // These keys are generated by the ReCaptcha service
	    publicKey = ""
	    privateKey = ""

	    // Include the noscript tags in the generated captcha
	    includeNoScript = true
	}

	mailhide {
	    publicKey = ""
	    privateKey = ""
	} 

	environments {
	  development {
	    recaptcha {
	      // Set to false to disable the display of captcha
	      enabled = false
	    }
	  }
	  production {
	    recaptcha {
	      // Set to false to disable the display of captcha
	      enabled = true
	    }
	  }
	}

## Externalized Configuration
See the Grails docs for examples of using externalized configuration files. The ReCaptcha config can be externalized as
the `.groovy` file (easiest), or it can be converted into a Java `.properties` file.

### Version 1.0 differences

The following configuration properties are no longer used and will be ignored if they are present in the config:

* `useSecureAPI` - All communications to the ReCaptcha servers are performed over HTTPS. There is no option to use HTTP.
* `forceLanguageInURL` - ReCaptcha now properly displays the captcha in the selected language. We no longer have to force the language.

# Usage - ReCaptcha

The plugin is simple to use. In order to use it, there are four basic steps:

## Edit the Configuration

The configuration values are pretty self-explanatory, and match with values used by the ReCaptcha service. You must enter your public and private ReCaptcha keys, or errors will be thrown when trying to display a captcha.

### Proxy Server Configuration

If your server needs to connect through a proxy to the ReCaptcha service, add the following to the ReCapctcha configuration. **These properties are not created by the quickstart script. They must be added manually.**

    proxy {
        server = ""   // IP or hostname of proxy server
        port = ""     // Proxy server port, defaults to 80
        username = "" // Optional username if proxy requires authentication
        password = "" // Optional password if proxy requires authentication
    }

Only the `server` property is required. The `port` will default to `80` if not specified. The `username` and `password` properties need to be specified only when the proxy requires authentication.

Like other configurations, this can be placed at the top-level `recaptcha` entry, or it can be specified on a per-environment basis.

## Use the Tag Library

The plugin includes four ReCaptcha tags:  `<recaptcha:ifEnabled>`, `<recaptcha:ifDisabled>`, `<recaptcha:recaptcha>`, and  `<recaptcha:ifFailed>`.

* The `<recaptcha:ifEnabled>` tag is a simple utility tag that will render the contents of the tag if the captcha is enabled in  `RecaptchaConfig.groovy`.
* The `<recaptcha:ifDisabled>` tag is a simple utility tag that will render the contents of the tag if the captcha is disabled in  `RecaptchaConfig.groovy`.
* The `<recaptcha:recaptcha>` tag is responsible for generating the correct HTML output to display the captcha. It supports three attributes: "theme", "lang", and "type". These attributes map directly to the values that can be set according to the ReCaptcha API. See the [ReCaptcha Client Guide](https://developers.google.com/recaptcha/docs/display#config) for more details.
* The `<recaptcha:ifFailed>` tag will render its contents if the previous validation failed. Some ReCaptcha themes, like "clean", do not display error messages and require the developer to show an error message. Use this tag if you're using one of these themes.

## Verify the Captcha

In your controller, call `recaptchaService.verifyAnswer(session, request.getRemoteAddr(), params)` to verify the answer provided by the user. This method will return true or false. Also note that `verifyAnswer` will return `true` if the plugin has been disabled in the configuration - this means you won't have to change your controller.

## Examples

Here's a simple example pulled from an account creation application.

### Tag Usage

In `create.gsp`, we add the code to show the captcha:

    <recaptcha:ifEnabled>
        <recaptcha:recaptcha theme="dark"/>
    </recaptcha:ifEnabled>

In this example, we're using ReCaptcha's "dark" theme. Leaving out the "theme" attribute will default the captcha to the "light" theme.

### Customizing the Language

If you want to change the language your captcha uses, set `lang = "someLang"` in the `<recaptcha/>` tag.

See [ReCaptcha Language Codes](https://developers.google.com/recaptcha/docs/language) for available languages.

### Verify User Input

Here's an abbreviated controller class that verifies the captcha value when a new user is saved:

	import com.megatome.grails.RecaptchaService
	class UserController {
		RecaptchaService recaptchaService

		def save = {
			def user = new User(params)
			...other validation...
			def recaptchaOK = true
			if (!recaptchaService.verifyAnswer(session, request.getRemoteAddr(), params)) {
				recaptchaOK = false
			}
			if(!user.hasErrors() && recaptchaOK && user.save()) {
				recaptchaService.cleanUp(session)
				...other account creation acivities...
				render(view:'showConfirmation',model:[user:user])
			}
			else {
				render(view:'create',model:[user:user])
			}
		}
	}


### Testing

Starting with version 0.4.5, the plugin should be easier to integrate into test scenarios. You can look at the test cases in the plugin itself, or you can implement something similar to:

	private void buildAndCheckAnswer(def postText, def expectedValid, def expectedErrorMessage) {
	    def mocker = new MockFor(Post.class)
	    mocker.demand.getQueryString(4..4) { new QueryString() }
	    mocker.demand.getText { postText }
	    mocker.use {
	      def response = recaptchaService.checkAnswer("123.123.123.123", "abcdefghijklmnop", "response")

	      assertTrue response.valid == expectedValid
	      assertEquals expectedErrorMessage, response.errorMessage
	    }
	}


The `postText` parameter represents the response from the ReCaptcha server. Here are examples of simulating success and failure results:

	public void testCheckAnswerSuccess() {
	    // ReCaptcha server will return true to indicate success
	    buildAndCheckAnswer("true", true, null)
	}

	public void testCheckAnswerFailure() {
	    // ReCaptcha server will return false, followed by the error message on a new line for failure
	    buildAndCheckAnswer("false\\nError Message", false, "Error Message")
	}


# Usage - Mailhide

## Edit the Configuration

The `recaptcha-quickstart` plugin creates basic configuration. You must enter your public and private Mailhide keys, or errors will be thrown when trying to display a Mailhide link.

## Use the Tag Library

The plugin includes two Mailhide tags: `<recaptcha:mailhide>` and `<recaptcha:mailhideURL>`.

* The `<recaptcha:mailhide>` tag creates a Mailhide URL that opens in a new, pop-up window per the Mailhide specification. It supports one attribute: "emailAddress", to specify the email to be hidden. The link will be created around whatever content is in the body of the tag.
* The `<recaptcha:mailhideURL>` tag creates a "raw" URL that can be used however desired. This is useful if the pop-up behavior of the other tag is not wanted. It supports two attributes: "emailAddress" and "var". The "emailAddress" attribute specifies the email to be hidden, while the "var" attribute specifies the name of the variable that the created URL should be available under in the page. The URL variable is only available in the context of the tag body.

## Examples

### mailhide tag

    <recaptcha:mailhide emailAddress="x@example.com">Some text to wrap in a link</recaptcha:mailhide>


will create:


	<a href="http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=..encryptedEmail..."
	     onclick="window.open('http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=...encryptedEmail...', '', 
	     'toolbar=0,scrollbars=0,location=0,statusbar=0,menubar=0,resizable=0,width=500,height=300'); return false;" 
	     title="Reveal this e-mail address">Some text to wrap in a link</a>


### mailhideURL tag

    <recaptcha:mailhideURL emailAddress="x@example.com" var="mu">
        Created Mailhide URL: ${mu}
    </recaptcha:mailhideURL>


will create:


    Created Mailhide URL: http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=...encryptedEmail...


# Misc.


### CHANGELOG

* 1.0.0 Initial support for the new "checkbox" style captcha. ([GitHub Issue #22](https://github.com/iamthechad/grails-recaptcha/issues/22))
    * This version of the plugin only supports the "traditional" captcha use case of automatic rendering. Explicit rendering will be available soon. See [the ReCaptcha docs](https://developers.google.com/recaptcha/docs/display) for more information about automatic vs. explicit.
    * `useSecureAPI` is no longer supported as a configuration option. All communication with ReCaptcha servers is over HTTPS.
    * `forceLanguageInURL` is no longer supported as a configuration option. ReCaptcha properly display the selected language no matter what language the browser uses.
* 0.7.0
    * Add support for connecting through a proxy server when verifying the captcha value. ([GitHub Issue #21](https://github.com/iamthechad/grails-recaptcha/issues/21))
* 0.6.9
    * Remove unused import for `org.codehaus.groovy.grails.commons.ConfigurationHolder` that doesn't exist in Grails 2.4 any more. ([GitHub Issue #20](https://github.com/iamthechad/grails-recaptcha/pull/20))
* 0.6.8
    * Don't crash when the `enabled` parameter is missing. Log missing config params, but use defaults. ([GitHub Issue #18](https://github.com/iamthechad/grails-recaptcha/issues/18))
    * Add blurb about externalized config. ([GitHub Issue #19](https://github.com/iamthechad/grails-recaptcha/issues/19))
* 0.6.7
    * Fix a stupid bug that would cause a crash when determining if it's enabled. ([GitHub Issue #14](https://github.com/iamthechad/grails-recaptcha/issues/14))
* 0.6.6
    * Behave correctly when config options are "false" or missing. ([GitHub Issue #13](https://github.com/iamthechad/grails-recaptcha/issues/13))
* 0.6.5
    * Don't crash when boolean configuration options are missing. ([GitHub Issue #10](https://github.com/iamthechad/grails-recaptcha/issues/10))
    * Establish defaults for boolean options in case they go missing. ([GitHub Issue #11](https://github.com/iamthechad/grails-recaptcha/issues/11))
    * Don't crash when creating an AJAX captcha. ([GitHub Issue #12](https://github.com/iamthechad/grails-recaptcha/issues/12))
* 0.6.4
    * Ensure that true/false settings are loaded correctly from a .properties file. ([GitHub Issue #9](https://github.com/iamthechad/grails-recaptcha/issues/9))
* 0.6.3
    * Ensure that AJAX tags properly use HTTPS when specified. ([GitHub Issue #7](https://github.com/iamthechad/grails-recaptcha/issues/7))
* 0.6.2
    * Remove spurious `println` left over. ([GitHub Issue #5](https://github.com/iamthechad/grails-recaptcha/issues/5))
    * Change install behavior to not create `RecaptchaConfig.groovy` in `_Install.groovy`. Add new script `recaptcha-quickstart` to handle creation of required configuration. ([GitHub Issue #6](https://github.com/iamthechad/grails-recaptcha/issues/6))
* 0.6.0
    * Add the ability to display the widget using AJAX. ([GitHub Issue #3](https://github.com/iamthechad/grails-recaptcha/issues/3))
    * Change plugin to require Grails 2.0 at a minimum.
* 0.5.3
    * Add the ability to force a different language to be displayed.
* 0.5.1 & 0.5.2
    * Update to use the new ReCaptcha URLs.
* 0.5.0
    * Add Mailhide support.
    * Add support for specifying configuration options elsewhere than `RecaptchaConfig.groovy` via the `grails.config.locations` method.
* 0.4.5
    * Add code to perform the ReCaptcha functionality - removed recaptcha4j library.
    * Don't add captcha instance to session to avoid serialization issues.
    * Hopefully make it easier to test against.
* 0.4
    * New version number for Grails 1.1. Same functionality as 0.3.2
* 0.3.2
    * Moved code into packages.
    * Tried to make licensing easier to discern.
    * Updated to Grails 1.0.4
* 0.3
    * Added support for environments and new `<recaptcha:ifFailed>` tag.
    * Updated to Grails 1.0.3
* 0.2
    * initial release, developed and tested against Grails 1.0.2

### Thanks

* The `recaptcha-quickstart` script was borrowed heavily from the [Spring Security Core plugin](http://grails.org/plugin/spring-security-core).


# Suggestions or Comments

Feel free to submit questions or comments to the Grails users mailing list.

[http://grails.org/Mailing+lists](http://grails.org/Mailing+lists)

Alternatively you can contact me directly - cjohnston at megatome dot com


[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/iamthechad/grails-recaptcha/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

