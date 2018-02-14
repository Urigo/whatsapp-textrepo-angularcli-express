# Step 3: Scaffolding

[//]: # (head-end)


Now we can start writing code!

But like we said before, software is build in layers and instead of starting from nothing, we can use existing software with prepared structure.

First let’s install some software that we need on our computer:

The following instructions are for computers with the Arch Linux operating system, so your mileage may vary.

First let's start by installing npm and node.js, as simple as `# pacman -S nodejs npm`.

Then we will need a way to install our npm global packages on a per-user basis, instead of relying on sudo: https://github.com/sindresorhus/guides/blob/master/npm-global-without-sudo.md

Now it's the right moment to install a couple of global packages we will need later on: `npm install -g @angular/cli node-sass tortilla typescript`

## IDE

Now it's time to choose our IDE. I suggest you Webstorm, but it's paid software with a kind-of perpetual EAP (Early Access Preview) available for free: https://blog.jetbrains.com/webstorm/category/eap/
If you decide to start using Webstorm keep in mind that sooner or later you'll have to start paying an annual subscription because EAPs are not always available.
To have a look at some of the Webstorm features you can have a look at those videos: https://www.jetbrains.com/webstorm/documentation/

Another, completely free (as in freedom) alternative is VSCode: https://code.visualstudio.com/
For most things it's as good as Webstorm, for others (like type inference) it's even better. Unfortunately it lacks most of the Webstorm features.

At the end the choice is yours.

## Mobile
We will talk once again about scaffolding once we will introduce `Android` and `Cordova`.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step2.md) | [Next Step >](step4.md) |
|:--------------------------------|--------------------------------:|

[}]: #
