# Psmb.Newsletter

With this package you can generate newsletters based on content of your Neos CMS website (e.g. digest of latest articles).

The cool part is that you render those newsletters using Fusion, which allows you to reuse any of your existing website Fusion objects and be completely flexible.

## Installation

`composer require "psmb/newsletter:@dev"`

## Configuration

Put something like this into your Settings.yaml:

```
Psmb:
  Newsletter:
    globalSettings:
      senderAddress: 'robot@server.com'
      senderName: 'Your robot'
    subscriptions:
      -
        identifier: daily
        renderer: 'Your.NameSpace:DailyLetterRenderer'
        label: 'Our daily newsletter'
        interval: P1D
      -
        identifier: weekly
        renderer: 'Your.NameSpace:WeeklyLetterRenderer'
        label: 'Our weekly newsletter'
        interval: P1W
```

Define as many subscription types under Psmb.Newsletter.subscriptions as you need. 

| setting key | description |
| --- | --- |
|`identifier`| Identifier of the subscription|
|`renderer`| Fusion object that would be used for rendering emails sent to this subscription. Defaults to `Psmb.Newsletter:MailRenderer`. Must inherit from it.|
|`interval`| Time interval identifier that would be used for selecting subscriptions. Can be used for cron jobs. Optional.|
|`senderName`, `senderAddress`| Override options from `globalSettings`.|

Also you may need to configure SwiftMailer, [see its docs](http://swiftmailer-for-flow.readthedocs.io/en/latest/#configuration) how to do that.

## Rendering newsletters

As said before, we render letters purely in Fusion, so it's completely up to you to define the rendering objects.
Entry point to rendering is `newsletter = Psmb.Newsletter:SubscriptionCase`, so you may hook straight into `Psmb.Newsletter:SubscriptionCase` to intercept the rendering process.
But the intended usage is to provide a different renderer per each subscription preset via the `renderer` setting key (see above).

Then define a Fusion object like this for each renderer:

```
prototype(Your.NameSpace:WeeklyLetterRenderer) < prototype(Psmb.Newsletter:MailRenderer) {
	subject = 'Our weekly digest'
	body = 'Generate message body. Use your imagination.'
  
  # You may automatically inline all css styles.
	# body.@process.cssToInline = Psmb.Newsletter:CssToInline {
	#	  cssPath = 'resource://Your.Package/Public/Path/To/Your/Styles.css'
	# }
  # # You may also override these settings, but usually no need to
  # format = 'plaintext' # defaults to 'html'
	# recipientAddress = ${subscriber.email}
	# recipientName = ${subscriber.name}
	# replyToAddress = ${subscription.senderAddress || globalSettings.senderAddress}
	# senderAddress = ${subscription.senderAddress || globalSettings.senderAddress}
	# senderName = ${subscription.senderName || globalSettings.senderName}
}
```

### You'd have the following context variables available:

|Context variable name|Description|
|---|---|
|`site`,`documentNode`,`node`|All point to `site` root node.|
|`subscriber`|`Subscriber` object. Has `name` and `email` keys.|
|`subscription`|Settings array for current subscription. `interval` key may be the most interesting one for generating lists of latest articles for given period.|
|`globalSettings`|Global settings array.|
|`activationLink`|Only available when generating email confirmation letters.|

### A more practical example:

```
prototype(Sfi.Site:DigestMail) < prototype(Psmb.Newsletter:MailRenderer) {
    # ElasticSearch used here, but could be a simple FlowQuery as well
    @context.nodes = ${Search.query(site).nodeType('Sfi.Site:News').exactMatch('type', 'ourNews').greaterThan('date', Date.format(Date.subtract(Date.now(), subscription.interval), "Y-m-d\TH:i:sP").sortDesc('date').execute()}
    @if.notEmpty = ${q(nodes).count() > 0}
    subject = ${Translation.translate('newsletter.digestSubject', null, [], null, 'Sfi.Site')}
    body = TYPO3.TypoScript:Collection {
        collection = ${nodes}
        itemName = 'node'
        itemRenderer = Sfi.Site:DigestArticle
    }
    @process.cssToInline = Psmb.Newsletter:CssToInline {
        cssPath = 'resource://Sfi.Site/Public/built/index.css'
    }
}

prototype(Sfi.Site:DigestArticle) < prototype(TYPO3.TypoScript:Tag) {
    tagName = 'a'
    attributes.href = NodeUri {
        node = ${node}
        absolute = ${true}
    }
    content = ${node.properties.title}
}
```

### Provided Fusion objects

|Object name|Description|
|---|---|
|`Psmb.Newsletter:MailRenderer`|Abstract mail renderer. Returns a RawArray of `body`, `subject` and other mail options passed to swiftmailer. See above for details.|
|`Psmb.Newsletter:EditSubscriptionLink`|Generated a link to edit action of the plugin|
|`Psmb.Newsletter:UnsubscribeLink`|Generated a link to unsubscribe action of the plugin|
|`Psmb.Newsletter:CssToInline`|Processor to automatically inline CSS styles|
|`Psmb.Newsletter:SubscriptionCase`|Rendering entry point. Usually no need to touch this one|
|`Psmb.Newsletter:SubscriptionPlugin`|Configured `TYPO3.Neos:Plugin` instance that insert Flow signup plugin.|

To customize the confirmation mail, override `Psmb.Newsletter:ConfirmationMailRenderer`.

## Rendering signup plugin

Insert `Psmb.Newsletter:SubscriptionPlugin` nodetype or render it manually.

Subscription has the following flow:

1. Fill in the sign up form. Asks for name, and email, and allows to choose between available subscription plans.
2. Confirm email.
3. Edit subscription options or unsubscribe from email links.

You absolutely have to override the default templates, the default ones are there just for demo purpose.
Create a `Views.yaml` file in Configuration:

```
-
  requestFilter: 'isPackage("Psmb.Newsletter")'
  options:
    templateRootPaths:
      - 'resource://Sfi.Site/Private/Newsletter/Templates/'
      - 'resource://Psmb.Newsletter/Private/Templates'
    partialRootPaths:
      - 'resource://Sfi.Site/Private/Newsletter/Partials/'
      - 'resource://Psmb.Newsletter/Private/Partials/'
    layoutRootPaths:
      - 'resource://Sfi.Site/Private/Newsletter/Layouts/'
```

## Acknowledgements

This is my first Flow package, and it wouldn't have been possible without a support of the community by answering dozens of n00b questions on Slack, by Christian Müller in particular.


