#  Fingerprint - Prevent sign-up fraud for Auth0 Users

The Fingerprint and Okta Customer Identity Cloud (CIC) powered by Auth0 integration is designed to provide a unique identifier for each of your user's devices. This is a powerful signal that helps reduce fraud and improve the user experience.

The integration is powered by Fingerprint Pro's device detection technology, which is an industry-leading solution that quickly and accurately identifies the characteristics of a browser or device. The device information, with unparalleled accuracy, is then used to create a unique and immutable device fingerprint that can be used to securely identify a user's device and detect any malicious activity.

Prerequisites
-------------

> __*Note*__: This integration requires the sign-up page to be hosted by the application so the Fingerprint library can be injected and the results sent as part of the sign-up request. This integration is not supported when using the Auth0 hosted sign-up page (Universal Login / classic Lock).

1.  An Auth0 account and tenant. [Sign up for free](https://auth0.com/signup).
2.  A Fingerprint Pro account. [Sign up for free](https://dashboard.fingerprint.com/signup/).
3.  A custom sign-up page (not Auth0 hosted / Universal Login). 

1\. Add Fingerprint Pro to your application
-------------------------------------------

To identify your visitors, add the Fingerprint Pro device intelligence agent to your website or application:

1.  [Sign up](https://dashboard.fingerprint.com/signup) for Fingerprint Pro if you haven't already and [log in](https://dashboard.fingerprint.com/login) to your dashboard.

2.  Navigate to **App Settings** → **Integrations** to explore the available SDKs and find the easiest way to install Fingerprint Pro.

3.  Add the SDK to your application - You can [import](https://dev.fingerprint.com/docs/js-agent#installing-the-agent--quick-usage-examples) the script directly in vanilla JavaScript or use a type-safe [SDK](https://dev.fingerprint.com/docs/frontend-libraries) for your favorite framework. Here is a [React SDK](https://github.com/fingerprintjs/fingerprintjs-pro-react) example wrapping the application (or component) within FpjsProvider:

    ```
    import {
         FpjsProvider,
         useVisitorData
     } from '@fingerprintjs/fingerprintjs-pro-react';

     const App = () => (
       <FpjsProvider
         loadOptions={{
           apiKey: 'YOUR_PUBLIC_API_KEY',
           endpoint: 'YOUR_CUSTOM_ENDPOINT'
         }}>
         <VisitorData />
       </FpjsProvider>
     );

     const VisitorData = () => {
       const { data } = useVisitorData();
       return (
         <div>
           Visitor ID: ${data?.visitorId}, Request ID: ${data?.requestId}
         </div>
       );
     };
    ```

    > The returned `visitorId` is a unique and stable identifier of your visitor.

    You can find your public API key in the Fingerprint dashbaord under **App Settings → API Keys**.



2\. Request Fingerprint on sign-up page and send results as part of Auth0 sign-up request
-----------------------------------------

> __*Note*__: This integration requires the sign-up page to be hosted by the application so the Fingerprint library can be injected and the results sent as part of the sign-up request. This integration is not supported when using the Auth0 hosted sign-up page (Universal Login / classic Lock).

1. Modify your sign-up page to include the Fingerprint API call and send the `visitorId` and `requestId` as additional sign-up parameters in the users `app_metadata`. 

    Below is a basic example in React which shows a sign-up form submit function making a request to the applications API component to create the user with the values captured on the sign-up form. In this case, the backend component proxies the request to the Auth0 Sign-up endpoint. 

 ```
 import React from "react";
 import { useVisitorData } from "@fingerprintjs/fingerprintjs-pro-react";

 const SignupComponent = () => {

  const { data } = useVisitorData();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const response = await fetch(`API_BASE_URL/signup`, {
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          "name": '',
          "email": '',
          "password": '',
          "fingerprint": `${data.visitorId} ${data.requestId}`,
        }),
        method: 'POST',
      });

      const responseData = await response.json();
      if (response?.ok) {
        //handle successful sign-up
      }
    } catch (err) {
      //handle err
    }
  };

  return (
    <Container className="signup-form">
      <form name="signup-form" onSubmit={handleSubmit}>
        {/* add form inputs */}
        <div class="sign-up-submit">
          <Button type="submit" variant="success" value="Submit" name="submit-form">Submit</Button>
        </div>
      </form>
    </Container>
  );
};
```

2. Then depending on your Auth0 implementation, add the following to your JSON data payload of the create user API request (Auth0 API):

  ```
    "app_metadata": { "signup_fingerprint": data.fingerprint }
  ```
  
  Your implementation details will vary depending on your user registration flow. The end result when creating a new user in Auth0 via API should be the fingerprint being sent within the users `app_metadata` as shown above.


3.  Open your website or application with Fingerprint Pro installed. You should see your identification event inside the Fingerprint [dashboard](https://dashboard.fingerprint.com/) → **Fingerprint Pro**.

Consult the Fingerprint Pro [Quick Start Guide](https://dev.fingerprint.com/docs/quick-start-guide) or contact [Fingerprint support](https://fingerprint.com/support/) if you have any questions.

> Note: For production deployments, we recommend routing requests to Fingerprint Pro APIs through your own domain. This prevents ad blockers from disrupting identification requests and improves accuracy. We offer a variety of proxy integration options, see [Protecting your JavaScript agent from ad blockers](https://dev.fingerprint.com/docs/protecting-the-javascript-agent-from-adblockers) for more details.


3\. Use the Fingerprint result in an Auth0 Action
-----------------------------------------------------

The Fingerprint `visitorId` and `requestId` will be available inside the Auth0 [Action](https://auth0.com/docs/customize/actions/actions-overview). The example action script below stores an array of visitorId's in the `app_metadata` of the users profile, checks the `visitorId` sent in the authorization params matches the `visitorId` for the associated request using Fingerprint's Event API and requests MFA as part of the authentication if the `VisitorId` (device/browser) is not recognised. 

1. Create a new [Pre User Registration Action](https://auth0.com/docs/customize/actions/flows-and-triggers/pre-user-registration-flow) in Auth0.

2. Use the example action script below in the Action to check for fraud on user sign-up.
  > The script prevents malicious actors creating multiple accounts on the same browser/device by checking to see if the assigned `visitorId` is stored against another user account. In this example, the `visitorIds` are stored in `app_metadata` of the users profile (within the Auth0 hosted database) - The script calls the Auth0 Management API to query the `app_metadata` for an existing `visitorId`, if a user account is returned, the sign-up is rejected as an account already exists for this device/browser. ***However, using `app_metadata` is not recommended for production, ideally the `visitorId` should be stored in your own user store as it's own attribute which can be queried via your API***.   
  
  > The script additionally checks the `visitorId` sent in the sign-up request matches the `visitorId` for the associated request using Fingerprint's [Event API](https://dev.fingerprint.com/reference/getevent).  The result of each [Smart Signal](https://dev.fingerprint.com/docs/smart-signals-overview) is also included in the `Event Identification` response which are checked for additional indicators of fraud such as bot detection.

3. Ensure that you replace the placeholder parameter values for `region` and `api_key` in the action script below. You can store the Fingerprint `api_key` within the [Auth0 secret values](https://auth0.com/docs/customize/actions/write-your-first-action#add-a-secret). 

4. Add the `@fingerprintjs/fingerprintjs-pro-server-api` library as a dependency of the Action using the [Auth0 Action Dependencies](https://auth0.com/docs/customize/actions/manage-dependencies).

```
/**
* Handler that will be called during the execution of a PreUserRegistration flow.
*
* @param {Event} event - Details about the context and user that is attempting to register.
* @param {PreUserRegistrationAPI} api - Interface whose methods can be used to change the behavior of the signup.
*/
exports.onExecutePreUserRegistration = async (event, api) => {
    //get fingerprint from sign-up request
    //in this example a value of signup_fingerprint is sent as part of the app_metadata
    const fingerprint = event.user.app_metadata?.signup_fingerprint;
    if (!fingerprint) {
        return;
    }
    var array = fingerprint.split(" ");
    const visitorId = array[0]
    const requestId = array[1]
    console.log("VisitorId is:" + visitorId);
    console.log("RequestId is:" + requestId);

    //spin up fingerpint client 
    const { FingerprintJsServerApiClient, Region } = require('@fingerprintjs/fingerprintjs-pro-server-api');
    // Init client with the given region and the secret api_key
    const client = new FingerprintJsServerApiClient({ region: Region.EU, apiKey: event.secrets.api_key });
    client.getEvent(requestId).then((event) => {
        var visIdCheck = event.products.identification.data.visitorId;
        console.log("Visitor Id tampering check: " + visIdCheck + " Visitor Id sent in request: " + visitorId)
        if (visIdCheck !== visitorId) {
            api.access.deny("tampering_detected", 'Sign-ups from this device cannot be accepted');
        }
        //optional - check smart signals here
        console.log("Bot Detected : " + event.products.botd.data.bot.result);
        if (event.products.botd.data.bot.result === "bad") {
            api.access.deny('bot_detected', 'Bot detected');
        }
        // console.log("Known to IP Blocklists : " + event.products.ipBlocklist.data.result);
        // console.log("Virtual Machine Detected : " + event.products.virtualMachine.data.result);
        // console.log("Tor Network Detected : " + event.products.tor.data.result);

    }).catch(function (err) {
        console.log(err)
    });
    //query existing users for same fingerprint
    //this could be using Auth0 app_metadata (not recommended for production) as shown below 
    //or using your own user store / API backend (recommended)

    //load Auth0 management client
    const ManagementClient = require('auth0').ManagementClient;
    const management = new ManagementClient({
        domain: event.secrets.domain,
        clientId: event.secrets.clientId,
        clientSecret: event.secrets.clientSecret,
    });
    var params = {
        search_engine: 'v3',
        q: 'app_metadata.visitorIds: ' + visitorId + '',
        per_page: 10,
        page: 0
    };
    try {
        const res = await management.getUsers(params)
        console.log("Count of users with same FP: " + res.length);
        // determine number of allowed accounts per device to allow
        if (res.length > 0) {
            api.access.deny("max_device_limit", 'Further Sign-ups from this device cannot be accepted');
        } else {
            api.user.setAppMetadata("visitorIds", [visitorId]);
        }
    } catch (e) {
        console.log(e)
        // Handle error - fail silently - don't block signup
    }

};
```

The above action script is just an example of how Fingerprint can prevent promo abuse or synthetic account creation at sign-up. See [Use cases](https://fingerprint.com/use-cases/) to explore different ways of preventing fraud and streamlining user experiences with Fingerprint.

Troubleshooting
---------------

To learn more about Fingerprint Pro, visit our [website](https://fingerprint.com/) or read the [documentation](https://dev.fingerprint.com/docs). If you have any questions, reach out to our [support](https://fingerprint.com/support/).