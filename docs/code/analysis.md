# Analysis

Tool for identifying problematic patterns found in JavaScript code.

- [ESLint](https://eslint.org/)
- [Sonar](https://www.sonarlint.org/)
- [JSCPD](https://github.com/kucherenko/jscpd/tree/master/packages/jscpd)
- [Husky](https://www.npmjs.com/package/husky)

## ESLint

### Installation

```bash
npm install eslint --save-dev
# or
yarn add eslint --dev
```

### Initialization

You should then set up a configuration file:

```bash
npx eslint --init
```

How would you like to use ESLint?

```bash
  To check syntax only
  To check syntax and find problems
❯ To check syntax, find problems, and enforce code style
```

What type of modules does your project use? (Use arrow keys)

```bash
❯ JavaScript modules (import/export)
  CommonJS (require/exports)
  None of these
```

Which framework does your project use?

```bash
  React
  Vue.js
❯ None of these
```

Does your project use TypeScript? (y/N)

``` bash
❯ Y
```

Where does your code run?

```bash
  ◯ Browser
❯ ◉ Node
```

How would you like to define a style for your project?

```bash
❯ Use a popular style guide
  Answer questions about your style
  Inspect your JavaScript file(s)
```

Which style guide do you want to follow?

```bash
❯ Airbnb
  Standard
  Google
```

What format do you want your config file to be in?

```bash
❯ JavaScript
  YAML
  JSON
```

Would you like to install them now with npm?

```bash
❯ Y
```


## JSCPD

Find duplicated blocks.

Copy/paste is a common technical debt on a lot of projects. The jscpd gives the ability to find duplicated blocks implemented on more than 150 programming languages and digital formats of documents. The jscpd tool implements Rabin-Karp algorithm for searching duplications.

- [Getting started](https://github.com/kucherenko/jscpd/tree/master/packages/jscpd#getting-started)

## Sonar

SonarLint is an IDE extension that helps you detect and fix quality issues as you write code.
Like a spell checker, SonarLint squiggles flaws so that they can be fixed before committing code.

- [vscode](https://www.sonarlint.org/vscode/);
- [intellij](https://www.sonarlint.org/intellij/)

## Husky

Husky can prevent bad git commit, git push and more 🐶 woof!

### Installation

```bash
npm install husky --save-dev
```

### Setup

Example:

```package.json
{
  "husky": {
    "hooks": {
      // start eslint on commit
      "pre-commit": "eslint api/**/*.js",
      // start test on push
      "pre-push": "npm run test",
    }
  }
}
```
