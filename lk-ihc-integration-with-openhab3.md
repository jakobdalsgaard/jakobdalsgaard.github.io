LK IHC integration with OpenHAB3

If you have a fairly old LK IHC controller in your house, then it is not able to use any of the TLS ciphers a recent Java may use. For security reasons these old ciphers have been disbled and thus OpenHAB 3 cannot talk to your controller. Usually, however, some old ciphers can be allowed by changing your configuration. On my Debian with Open JDK 11 this is done by editing `/etc/java-11-openjdk/security/java.security` -- find the line starting with `jdk.tls.disabledAlgorithms=` and remove TLSv1 from the list. Then restart openhab service by doing `sudo systemctl restart openhab.service` -- and voilà your OpenHAB 3 should be able to connect to your old LK IHC controller.

Tags: house, java, security
