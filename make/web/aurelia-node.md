

Although this article is a repeat of a lot of what is said on the [aurelia-node github page](https://github.com/zewa666/aurelia-node), I find it useful to rewrite instructions as it helps me remember and clarify things.

# Install

This article assumes that you've already installed git, gulp, node, npm, jspm, etc.  If you haven't done so then please do that first.

## Download aurelia-node

Make a directory for your project and navigate to it.

```git clone https://github.com/zewa666/aurelia-node.git .```

## Install Dependencies

Install the node dependencies.  The node dependencies are specified in the **package.json** file that is specified in the aurelia-node git repository.
```
npm install
```

Install the skeleton app by downloading it from github.
```
node bin/install
```

Install the JavaScript dependencies.
```
cd public/app
jspm install
```
