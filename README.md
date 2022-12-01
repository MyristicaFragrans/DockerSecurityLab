# Docker Security Lab
This is a set of demos meant to walk you through different exploits and
potential vulnerabilities in Docker (and other container) runtimes, then
describe how to resolve the issue.

Download this repository with
```
git clone https://github.com/MyristicaFragrans/DockerSecurityLab
```

## Prerequisites
1. Basic Docker experience
2. Basic Linux Commands
3. You need [Vagrant](vagrantup.com/) installed. Vagrant will handle all other
installations and prerequisites.

## 01: Binds Escalation
Go to [Binds Escalation](01_Binds_Escalation/Binds_Escalation.md) to learn how
to use Docker's `--mount` to gain root access to a machine.

## Continuing
I will add more demos and examples as I discover them