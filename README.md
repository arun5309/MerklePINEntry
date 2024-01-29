# Introduction

This is a prototype for a [Merkle's puzzles](https://en.wikipedia.org/wiki/Merkle's_Puzzles) based PIN entry method that is resistant to over-the-shoulder attacks. Instead of entering the PIN directly the user enters the PIN one digit at a time while solving a Merkle's puzzle chosen arbitrarily by the user (here the puzzle is addition modulo 100. The first digit serves as the identifier and the second digit acts as the value to shift (addition modulo 10) the PIN digit). During the entry phase for each digit the user needs to type the identifier digit followed by the shifted value. The attacker who doesn't know the entry picked by the user is unable to guess the password.

# Usage

Run the below command to install dependencies (Node JS and npm are expected to be installed on your system). Other package managers like yarn, pnpm work too if they are installed.

```sh
$ npm install
```

Run the below command to build the program once the dependencies are installed.

```sh
$ npm run build
```

Run the below command to run the program once the dependencies are installed. Can be accessed in the browser (running only on the host device) on the URL localhost:4173 or {local IP of host device}:4173.

```sh
$ npm run preview_host
```
