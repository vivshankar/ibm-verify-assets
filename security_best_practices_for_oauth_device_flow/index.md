# Security best practices for deploying the OAuth 2.0 Device Flow with IBM Verify

The OAuth 2.0 Device Authorization Grant Flow (or simply Device Flow) enables devices with no browser or limited input capability, such as smart devices like Amazon Echo, to obtain an access token that can be used by the device to access protected resource APIs on behalf of a user. This is commonly seen on applications on Apple TV, Android TV, Amazon Echo and others where a user is shown a QR code or link including an optional one-time registration code. The user logs in using a companion device, like a smart phone or laptop.

The device that requires the token is called a _consuming device_ and the one used by the user to authenticate and authorize the request is called an _authorizing device_. These terms will be used in this article.

Depending on how the OAuth 2.0 client is configured, it can be prone to phishing attacks, as described in this article titled [OAuth’s Device Code Flow Abused in Phishing Attacks](https://www.secureworks.com/blog/oauths-device-code-flow-abused-in-phishing-attacks).

IBM Verify offers mechanisms to apply security best practices to protect this flow.

## The Device Flow in IBM Verify

IBM Verify allows clients to be configured without client secrets, i.e. they can be public clients with only a `client_id`. This is typically used when the OAuth flow is initiated and completed on an untrusted front-end (for example a browser-based single page application) or is compiled into code that is distributed and hosted by various parties as part of another software product (a mobile application or mobile SDK).

The Device Flow begins with a call to `/oauth2/device_authorization` from the consuming device. Notice in this example that a public OAuth 2.0 client is used.

```curl
curl 'https://abc.verify.ibm.com/oauth2/device_authorization' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=device_client_id'
```

The response is as below:

```json
{
    "device_code": "tKcKfPUA_hF1CIqS-uL1TxlHHg84qpVt1ssIWw7q6-E.eHPgVHStl7sG-GGN9M8GD8omTz38vavHT19idT5uKspT1O0K0k00bsyxIhNow4Y_2Ij1SEPTzyDV7t_v8BG9NQ",
    "expires_in": 299,
    "interval": 5,
    "user_code": "70HYPR",
    "verification_uri": "https://abc.verify.ibm.com/oauth2/user_authorization",
    "verification_uri_complete": "https://abc.verify.ibm.com/oauth2/user_authorization?user_code=38ORIH"
}
```

The device keeps track of the `device_code` and displays the `user_code` and either the `verification_uri` or the `verification_uri_complete`. Note in the diagram below that the `verification_uri` has been shortened using a custom URL shortener. The `verification_uri_complete` might also be shown as a QR code if the intent is for the user to scan it with a mobile phone and use their phone's browser to complete the next step.

![](device_display.png)

When the user scans the QR code or browses to the link on a browser, the user is asked to enter the `user_code` value. It may be prefilled or the prompt skipped entirely when the QR encodes `verification_uri_complete` where the `user_code` is included as a query string parameter.

![](user_authorization_enter_user_code.png)

Once the user enters the user code, the user is taken through a login process and authorizes the request to proceed. Authentication can include the use of phishing resistant methods like passkeys and include adaptive MFA policies. Note that while user authentication on their browser can (and is recommended to) be phishing resistant, this does not make the entire device code flow phishing resistant as there is no guarantee that the `user_code` isn't that of a bad actor.

![](user_sign_in_page.png)

While the user is busy on their `authorization device` completeing this process, the `consuming device` will poll for completion by invoking the token endpoint:

```curl
curl 'https://abc.verify.ibm.com/oauth2/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=device_client_id' \
--data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:device_code' \
--data-urlencode 'device_code=tKcKfPUA_hF1CIqS-uL1TxlHHg84qpVt1ssIWw7q6-E.eHPgVHStl7sG-GGN9M8GD8omTz38vavHT19idT5uKspT1O0K0k00bsyxIhNow4Y_2Ij1SEPTzyDV7t_v8BG9NQ'
```
> 📘 Note
> 
> While the example above shows the use of a public client, IBM Verify offers multiple client authentication methods, including the use of client secrets, JWTs, client certifications etc. The public client is used here to demonstrate and describe best practices related to the phishing attack, which is the purpose of this article.

When the user completes the sign in process, the token call returns a valid access token that can then be used to call protected APIs from the consuming device.

## The attack

With the use of a public OAuth 2.0 client, it is safe to assume the `client_id` is known to the attacker. The adversary can make the API call to the `device_authorization` endpoint to obtain a valid set of device and user codes, including the `verification_uri_complete`. The adversary now uses standard phishing techniques - a legitimate looking email etc. - to convince the victim to visit the attacker's `verification_uri_complete` URI and complete authorization. When the victim does this, the next poll to the token endpoint by the adversary results in legitimate access/refresh tokens that can be used to access protected resources on behalf of the victim.

Notice here that the user may use the strongest form of authentication on their `authorizing device` and is still prone to this form of attack.

## Mitigations

There are several strategies that may be employed to help mitigate this form of an attack. Many of these are discussed in detail in [Cross-Device Flows: Security Best Current Practice](https://datatracker.ietf.org/doc/draft-ietf-oauth-cross-device-security/) (draft 10 at time of writing). In this section we are going to highlight how IBM Verify may be configured to implement best practices for User Experience (secton 6.1.14 of draft 10), in particular for displaying context to the user during authorization consent.

### Add a consent page that provides additional context to the user

The [device flow standard](https://www.rfc-editor.org/rfc/rfc8628.html#section-5.4) recommends the following.

![](rfc_phishing_snippet.png)

You can accomplish this in IBM Verify by displaying a consent page that shows the user code and provides information, such as the following.

![](user_consent.png)

In order to achieve this experience, follow the steps as below.

![](config_steps_for_consent.png)

#### Configure the Attribute representing the user code

1. Go to Directory > Attributes on the IBM Verify Admin Console.

    ![](attributes-01-menu.png)

2. Begin the process of adding a fixed value attribute. This will act as a placeholder representing the user code.

    ![](attributes-02-add-01.png)

3. Name the attribute "User Code" with a specific identifier called "user_code". The ID is important because this is used later.

    ![](attributes-03-add-02.png)

4. Set a default value. This value is never used.

    ![](attributes-04-add-03.png)

5. Attribute has been added.

    ![](attributes-05-add-04.png)

#### Configure the Data Privacy purpose representing the user approval

1. Go to Data privacy and consent > Data purposes

    ![](purposes-01-menu.png)

2. Begin the process to create the purpose. The purpose ID must be set to `user_code_approval`. This value is used later.

    ![](purposes-02-add-01.png)

3. Choose the default access type.

    ![](purposes-02-add-02.png)

4. Add the user code attribute to the purpose.

    ![](purposes-02-add-03.png)

5. Publish the data purpose.

    ![](purposes-02-add-04.png)

6. The purpose has been created.

    ![](purposes-02-add-05.png)

#### Update the application configured to use device flow

The expectation is the application has already been configured to use device flow and is using the OpenID Connect application type from the catalog.

1. Navigate to the application and edit. Go to the "Sign on" tab.

2. Scroll down to the Endpoint Configuration section and edit the Authorize endpoint settings. This will pop a modal dialog up.

    ![](apps-01-consent-request-1.png)

3. Edit the Consent Request.

    ![](apps-01-consent-request-2.png)

4. Copy the expression below and paste it in the code editor. This uses the CELx language that IBM Verify uses for scripting safely.

    ```yaml
    statements:
    # If the user code is present in the request, add it to the consent request items.
    - context: >
        user_code_scope := requestContext.getValue("user_code") != "" ? [{
            "purpose": "user_code_approval",
            "attribute": "user_code",
            "value": requestContext.getValue("user_code")
        }] : []
    # Auto-authorize all requested OAuth scopes.
    # If you don't want the client requesting
    # any OAuth scopes, just drop this entire statement and return
    # the context.user_code_scope.
    - context: >
        auto_grant_scopes := has(requestContext.scope) ? requestContext.scope.map(x, {
            "purpose": "ibm-oauth-scope",
            "attribute": x,
            "scope": x,
            "autoGrant": true
        }) : []
    - return: >
        context.user_code_scope + context.auto_grant_scopes
    ```

    The editor will look as below. Click OK.

    ![](apps-01-consent-request-3.png)

5. Scroll down in the "Sign on" tab and ensure the consent action is configured to ask for consent.

    ![](apps-01-consent-request-4.png)

6. Save the application.

7. Go to the "Privacy" tab and add a purpose.

    ![](apps-02-privacy-1.png)

8. Add the "User code approval" purpose to the list of allowed purposes for this application.

    ![](apps-02-privacy-2.png)

9. Save the application.

    ![](apps-02-privacy-3.png)

#### Update the consent page template

The consent page is a customizable HTML template in a theme.

1. Go to "User experience" > "Branding".

    ![](branding-01-menu.png)

2. Choose the appropriate theme that is associated with your application.

    ![](branding-02-default.png)

3. Navigate to the OIDC user consent page and edit. This is illustrated in the picture.

    ![](branding-03-tree.png)

4. Above the "consent_form_verifier" hidden field and the submit button, add the following snippet. You may re-style this as needed. This segment shows the code to the user and provides any appropriate instructions.

    ```html
    [RPT purpose_user_code_authorization]
    <div class="bx--form-item">
        <p>Verify that the code you see here matches the one on the app.</p>
        <div style="font-size:14pt">
        @PRIVACY_SCOPE_ATTRVALUE_REPEAT@
        </div>
    </div>
    [ERPT purpose_user_code_authorization]
    ```

5. Click Save.

#### Run the flow

You can now repeat the device flow and you will notice the additional consent page that shows the user the code and instructions to compare it to the one displayed on the application. Adjust the wording as appropriate.

## The wrap

This article introduced some of the security implications of using the OAuth 2.0 Device Authorization Grant flow, in particular, with public clients. While the use of public clients may be unavoidable in certain situations, adding additional context for a user to make a decision helps improve the security posture.
