# Building safeposix-rust

## Setting environment

We need to set environment with the following codes

```
cd /home
apt update
apt install git
apt install curl
apt install gcc
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs/ | sh
rustup install nightly
. "$HOME/.cargo/env"
rustup default nightly
```

## Clone git repo

Then we clone it to home directory
```
git clone --recurse-submodules https://github.com/yzhang71/safeposix-rust.git
```

Switch branch to 3i-dev

```
cd safeposix-rust
git switch 3i-dev
```

## Build

Compile and make sure there are librustposix.so

```
cargo build
```
