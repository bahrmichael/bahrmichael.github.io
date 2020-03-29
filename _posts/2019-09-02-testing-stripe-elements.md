---
layout: post
title: Testing Stripe Elements with Cypress
---

In this article we will look into how we can test a website that uses iframes like Stripe Elements.

We recently started using [Cypress](https://www.cypress.io/) as our E2E testing framework and love it for its simplicity and ease of use. If you haven’t used it before [give it a try](https://www.cypress.io/)!

While Cypress is elegant to work with on your own code, it might get tricky when you use third party components in your webapp. In our case the users can pay with credit card. To collect the payment details we use [Stripe Elements](https://stripe.com/docs/web/setup) which loads an iframe into our website. Cypress however [doesn’t support iframes yet](https://docs.cypress.io/guides/references/known-issues.html#Iframes). Luckily [mightyiam](https://github.com/mightyiam) found a [workaround](https://github.com/cypress-io/cypress/issues/136#issuecomment-525994895) to access iframes and their contents with Cypress.

**Setup**

We’re assuming that you set up a Cypress project and ran the example test suite. Your project structure should look like the picture below.

![Project structure](https://cdn-images-1.medium.com/max/2000/1*2c0K0Iu7nXsrsEvOiwheww.png)

We will start with extending the commands.js file.

<iframe src="https://medium.com/media/37836aafe54b81c1a55eb3f452d5c991" frameborder=0></iframe>

Append this code to your commands.js file. Now you can use cy.getWithinIframe('a selector') to target any element within the iframe.

<iframe src="https://medium.com/media/a6af7bed4fd8d4a8b9bb76884faf0ab2" frameborder=0></iframe>

If you need greater flexibility you can use iframeLoaded and getInDocument as shown on the left. You can chain many Cypress commands after getInDocument.

**Typing into Stripe Elements**

Now we’re able to fill out credit card details. In your browser’s [dev tools](https://developers.google.com/web/tools/chrome-devtools/) you can see the structure of the iframe being loaded by Stripe Elements.

![Iframe structure of Stripe Elements](https://cdn-images-1.medium.com/max/3300/1*uyKds_NEMJzxz3uJqJuE2Q.png)

As you can see in the highlighted area, there’s an input that we can select with [name="cardnumber"]. With this information we can tell Cypress to fill out the test credit card number 4242 4242 4242 4242 as well as the other required information. The other input parts are exp-date, cvc and postal.

<iframe src="https://medium.com/media/60396f0258a3ad4f10d47665718634ec" frameborder=0></iframe>

Integrate the code into your test suite and run it. You should see Cypress filling out the credit details.

![Cypress fills out credit card details](https://cdn-images-1.medium.com/max/4188/1*d3CSyLxOKn8LH5OxVgVHRw.gif)

That’s it! You can now test your complete order process without any hacks.

**Troubleshooting**

If you’re running Cypress with Chrome you might have to disable [web security](https://docs.cypress.io/guides/guides/web-security.html#Limitations) by adding "chromeWebSecurity": false to the cypress.json file.

**Open issues**

We haven’t tested how this behaves with multiple iframes.