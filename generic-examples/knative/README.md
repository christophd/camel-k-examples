# Knative Camel K Example

This example shows how Camel K can be used to connect Knative building blocks to create awesome applications.

A video version of this [demo is available on YouTube](https://youtu.be/btf_e2GniXM).
**NB:** This video shows an outdated way of configuring the Camel Telegram component. The recent way uses a required `authorizationToken` query parameter (see the "Playing harder" section below).

It's assumed that both Camel K and Knative are properly installed (both Knative Serving and Eventing) into the cluster. 
If not, please refer to the [official documentation to install and configure the components](https://knative.dev/docs/install/).

We're going to create two channels:
- messages
- words

The first channel will contain phrases, while the second one will contain the single words contained in the phrases.

To create the channels (they use the in-memory channel provisioner):

```
kubectl create -f messages-channel.yaml
kubectl create -f words-channel.yaml
```

We can now proceed to install all camel K integrations.

## Install a "Printer"

We'll install a Camel K integration that will print all words from the `words` channel.

Writing a "function" that does this is as simple as writing:

```yaml
- from:
   uri: "knative:channel/words"
   steps:
    - to: "log:info"
```

You can run this integration by running:

```shell
kamel run --dev printer.yaml
```

Under the hood, the Camel K operator does this:
- Understands that the integration is passive, meaning that it can be activated only using an external HTTP call (the knative consumer endpoint)
- Materializes the integration as a Knative autoscaling service, integrated in the Istio service mesh
- Adds a Knative Eventing `Subscription` that points to the autoscaling service

The resulting integration will be scaled to 0 when not used (if you wait ~5 minutes, you'll see it).

## Install a "Splitter"

We're now going to deploy a splitter, using the Camel core Split EIP. The splitter will take all messages from the `messages` channel,
split them and push the single words into the `words` channel.

The integration code is super simple:

```yaml
- from:
    uri: "knative:channel/messages"
    steps:
      - split:
          tokenize:
            token: ' '
          steps:
            - log: "sending ${body} to words channel"
            - to: "knative:channel/words"
```

Let's run it with:

```shell
kamel run --dev splitter.yaml
```

This integration will be also materialized as a Knative autoscaling service, because the only entrypoint is passive (waits for a push notification).

## Install a "Feed"

We're going to feed this chain of functions using a timed feed like this:

```yaml
- from:
   uri: "timer:clock?period=3000"
   steps:
    - setBody:
       constant: "Hello World from Camel K"
    - to: "knative:channel/messages"
    - log: "sent message to channel: messages"
```

Every 3 seconds, the integration sends a message to the Knative `messages` channel.

Let's run it with:

```shell
kamel run --dev feed.yaml
```

This cannot be materialized into an autoscaling service, but the operator understands it automatically and maps it to a plain Kubernetes Deployment
(Istio sidecar will be injected).

## Playing around

If you've installed all the services, you'll find that the printer pod will print single words as they arrive from the feed (every 3 seconds, passing by the splitter function).

If you now stop the feed integration (`kamel delete feed`) you will notice that the other services (splitter and printer) will scale down to 0 in few minutes.

And if you reinstall the feed again (`kamel run feed.yaml`), the other integration will scale up again as soon as they receive messages (splitter first, then printer).

## Playing harder

You can also play with different kind of feeds. E.g. the following simple feed can be used to bind messages from Telegram to the system:

```
from('telegram:bots?authorizationToken=<put-your-botfather-authorization-token-here>')
  .convertBodyTo(String.class)
  .to('log:info')
  .to('knative:channel/messages')
```

Now just send messages to your bot with the Telegram client app to see all single words appearing in the printer service.

You can also replace the printer with a Slack-based printer like:

```
from('knative:channel/words')
  .log('Received: ${body}')
  .to('slack:#camel-k-tests')


camel {
  components {
    slack {
      webhookUrl '<put-your-slack-incoming-webhook-url-here>'
    }
  }
}
```

Now the single words will be printed in the log but also forwarded to the
slack channel named `#camel-k-tests`.

You have infinite possibilities with Camel!
