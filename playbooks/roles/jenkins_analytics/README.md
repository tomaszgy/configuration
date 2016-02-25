Jenkins Analytics
=========

A role that sets-up Jenkins for deploying analytics stack
on AWS EMR.

It does following steps:

* Installs jenkins using ``jenkins_master``
* Configures ``config.xml`` to enable security and use
  Linux Auth Domain.
* Enable the use of jenkins clis

Configuration
------------

Jenkins auth domain is configured as such:

    jenkins_auth_realm:
      name: unix
      service: su
      plain_password: jenkins
      crypted_password: $6$rAVyI.p2wXVDKk5w$y0G1MQehmHtvaPgdtbrnvAsBqYQ99g939vxrdLXtPQCh/e7GJVwbnqIKZpve8EcMLTtq.7sZwTBYV9Tdjgf1k.

In future we will try enable other auth domains,while
preserving ability to run cli.


Dependencies
------------

- jenkins_master
