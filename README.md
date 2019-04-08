# Quarterly Deep Dive Enablement demo

This demo shows deployment of AMQ Streams 1.1 on OpenShift togerther with a simple demo application which demonstrates different common patterns used with Apache Kafka.

The Java source codes for the demo can be found [here](http://todo).

## Create Projects / namespaces

* Create new projects for the demo
```
oc new-project amq-streams-operator
oc new-project amq-streams
oc new-project amq-streams-demo
```

## Install AMQ Streams operator

* Switch to the project `amq-streams-operator`
```
oc project amq-streams-operator
```

* Install AMQ Streams 1.1
```
oc apply -f 01-install/
```

## Deploy Kafka

* Switch to the project `amq-streams`
```
oc project amq-streams
```

* Install Kafka cluster
```
oc apply -f 02-kafka.yaml -n amq-streams
```

* Copy the broker service key to the `amq-streams-demo` namespace
```
oc get secret my-cluster-cluster-ca-cert --namespace=amq-streams --export -o yaml | oc apply --namespace=amq-streams-demo -f -
```

## Deploy the price feed

The price feed here is just a demo. It generates random prices for selected stock tickers.

* Switch to the project `amq-streams-demo`
```
oc project amq-streams-demo
```

* Deploy the price feed generator and a simple UI for monitoring the prices
```
oc apply -f 03-price-feed.yaml -n amq-streams-demo
```

* Check that the pods are working and use to UI to see how the prices are generated.

## Deploy the trade manager

The trade manager will let you enter trades.

* Switch to the project `amq-streams-demo`
```
oc project amq-streams-demo
```

* Deploy the price feed generator and a simple UI for monitoring the prices
```
oc apply -f 04-trade-manager.yaml -n amq-streams-demo
```

* Check that the pods are working and use to UI to enter new trades. You can use negative amounts to sell shares.

## Deploy the portfolio aggregator

The portfolio aggregator will take the trades you created and aggregate them into a portfolio. 
A portfolio will give you a clear view of what you own.

* Switch to the project `amq-streams-demo`
```
oc project amq-streams-demo
```

* Deploy the price feed generator and a simple UI for monitoring the prices
```
oc apply -f 05-portfolio.yaml -n amq-streams-demo
```

* Check that the pods are working and use to UI to check your portfolio. Try to enter new trades in the UI from the previous step to see how the portfolio is updated.

## Deploy the pricer

The pricer will price the portfolio. 
It will join the stream with prices with the stream with portfolios and tell you how much are the positions in your portfolio worth.

* Switch to the project `amq-streams-demo`
```
oc project amq-streams-demo
```

* Deploy the price feed generator and a simple UI for monitoring the prices
```
oc apply -f 06-pricer.yaml -n amq-streams-demo
```

* Check that the pods are working and use to pricer UI to see the priced portfolio
