# Node.js Notes
This is a list of Node.js notes collected while try to figure out stuff about the subject

## Setup

#### How to download and install
From https://nodejs.org/en/ and try:
```
$ node --version
$ npm --version
```

#### What the scope of npm packages?
When installing packages with the npm install command, you have to decide whether you’re adding them to your current project or installing them globally.

#### How to use it? 
```
$ npm init -y <- cerate the package.json
$ npm i --save express <- install a library express and save to package.json
$ npm hello.js
$ node debug hello.js
$ node --inspect --debug-brk
```

#### How to create a module?
a) just create a file mymodule.js or b)Create a directory, init, and generate an index.js

#### How to expose a function from a module? And use it?
```
exports.canadianToUS = canadian => roundTwo(canadian * canadianDollar); <- expose
...
const currency = require('./currency'); <- to use it
console.log(currency.canadianToUS(50));
```

#### How to expose a Class from a module? And use it?
Adding module. to exports
```
lass Currency {
  constructor(canadianDollar) {
   this.canadianDollar = canadianDollar; 
  }
...
module.exports = exports = Currency;
....

const Currency = require('./currency'); <- to use it
const canadianDollar = 0.91;
const currency = new Currency(canadianDollar);
console.log(currency.canadianToUS(50));
```


