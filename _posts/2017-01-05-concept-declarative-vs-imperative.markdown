---
layout: post
title:  "Quick note on declarative vs imperative programming"
date:   2017-01-05 17:19:27 +0700
---

**Imperative** programming is when you tell computer to do exact thing step by step.
The computer won't use their own specified/trained logic, 
and directly trust with your command that are given as lines of code.

Your code will likely be very explicit and detailed.
Missing one line/command will usually lead you to error.

#### Example in the real world

If you want to go to the mall by taxi, you may want to tell where your driver would go on each intersection.

```
Please go left here.
(reaches another intersection)
Please go right here here.
(reaches another intersection)
Please go straight here.

... and so on
```

#### Example in coding

Supposed you want to create automated script that open certain website, 
type some search keyword, submit the form, and get all the search results.

You will then need to write the code to do exact thing step by step.

```
// open www.example.com
// get the input field by certain id "x"
// type some search keyword inside the input field
// get submit button by certain id "y"
// press the submit button
// wait until page finished loading
// retrieve the search results
```

<br>
<br>
<br>

On the contrary, **Declarative** Programming is when you "declare" to computer what you want in the end state, 
and let them do their own way to fulfill your request.

The computer/tech/stack will usually have their own way of logic/rule/priorities in how they 
handle your request.

Your code will likely be implicit and [idempotence](https://en.wikipedia.org/wiki/Idempotence){:target="_blank"}:
running the script/code once or multiple time during the process will lead to the same single outcome.


#### Example in the real world

If you want to go to the mall by taxi, you may want to only tell the driver to:

```
"Please go to Lincoln Road Mall"
```

The driver will figure the rest,
and may take a different route than you had anticipated,
but the end state will be the same.


You can say that sentence 15 times during the driving, and the end state will just be the same.
Though the driver will probably get a bit irritated.


#### Example in coding

If we want to spin up an NGINX deployment that has 3 replica pods inside a kubernetes cluster,
we only need to tell kubernetes "what we want" inside the kubernetes yaml config.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

We can run the script once, multiple time during spinning up process, 
por even running multiple time after the whole stack finished & ready,
the end state will always be the same.
