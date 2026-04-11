# systemd-hardene.d

> A community-driven systemd security guide and collection of hardened service configurations

## Motivation

Most of software developers/packagers don't propose any proper confinement (it's pretty time-consuming and usually situational, hard to come to a profile that could suit 100% of cases without breakage).

This project aims at :

* rigorously listing systemd security-related options with their corresponding documentation (see [`systemd_service_hardening.md`](docs/systemd_service_hardening.md)) for software developers and system administrators

* providing a collection of "out-of-the-box" systemd service unit hardening configuration profiles for well-known services running in Linux environments (see [`collection/`](collection/))

## Requirements

* systemd

## Installation

For instance, hardening Gitea systemd service is easy as :

```bash
## 1. Clone this project
git clone -C /usr/local/share https://github.com/HorlogeSkynet/systemd-hardene.d

## 2. Prepare Gitea systemd service unit override directory tree
mkdir -p /usr/local/lib/systemd/system
ln -s /usr/local/share/systemd-hardene.d/collection/gitea.service.d /usr/local/lib/systemd/system

## 3. Reload systemd configuration and restart Gitea
systemctl daemon-reload
systemctl restart gitea.service
```

## Frequently Asked Questions

### I don't like your security profile and/or I've got a specific use-case which isn't covered, what should I do ?

> Sure, this project cannot fit _everyone_ needs. Service unit overrides are shipped as `00-hardened.conf`, and one could easily extend it by creating another unit override which would be lexicographically-loaded afterwards (e.g. `/etc/systemd/system/fixme.service.d/01-relaxed.conf`). Simply run `systemctl edit fixme.service` and add your own override this way.

### Do you support systemd user instances ?

> You can try to set this service unit overrides collection to be sourced by systemd user instances, by replacing `/usr/local/lib/systemd/system` by `/usr/local/lib/systemd/user` during install steps. Please note that many security items **only** apply to system services and thus additional relaxed overrides may be required.

### I use some security overrides for several years, can I contribute ?

> Absolutely ! Simply open up a pull request including your profiles. Please try to honor existing service unit overrides style and format for project consistency :-)

## Acknowledgments

* [@ageis](https://github.com/ageis)'s original [systemd_service_hardening.md](https://gist.github.com/ageis/f5595e59b1cddb1513d1b425a323db04)
* [@krathalan](https://github.com/krathalan)'s [systemd-sandboxing](https://github.com/krathalan/systemd-sandboxing)
* [Keep Your Sandbox Tight!](https://github.com/rusty-snake/kyst/tree/main/systemd)

## Related Projects

* [SHH](https://github.com/desbma/shh) (Systemd Hardening Helper), for a rather dynamic/exhaustive/situational approach
* [@pgerber](https://github.com/pgerber)'s [systemd hardening guide](https://docs.arbitrary.ch/security/systemd.html)
* [Systemd Units Hardening](https://docs.rockylinux.org/10/guides/security/systemd_hardening/)
