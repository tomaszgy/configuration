# Jenkins Analytics

A role that sets up Jenkins for scheduling analytics tasks. 

This role performs the following steps:

* Installs Jenkins using `jenkins_master`.
* Configures `config.xml` to enable security and use
  Linux Auth Domain.
* Creates Jenkins credentials. 
* Enables the use of Jenkins CLI. 
* Installs a seed job from configured repository, launches it and waits
  for it to finish.

## Configuration

When you are using vagrant you **need** to set `VAGRANT_JENKINS_LOCAL_VARS_FILE`
environment variable. This variable must point to a file containing 
all required variables from this section.

This file needs to contain, at least, the following variables 
(see the next few sections for more information about them): 

* `JENKINS_ANALYTICS_USER_PASSWORD_HASHED` 
* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`
* `JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID` or `JENKINS_ANALYTICS_CREDENTIALS`

 
### End-user editable configuration 

#### Jenkins user password

You'll need to override default `jenkins` user password, please do that
as this sets up the **shell** password for this user. 

You'll need to set both a plain password and a hashed one.
To obtain a hashed password use the `mkpasswd` command, for example: 
`mkpasswd --method=sha-512`. (Note: a hashed password is required 
to have clean "changed"/"unchanged" notification for this step 
in Ansible.) 

* `JENKINS_ANALYTICS_USER_PASSWORD_HASHED`: hashed password 
* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`: plain password

#### Jenkins seed job configuration 

This role creates a seed job in Jenkins for every enabled analytics task.  By
default, 5 analytics tasks are enabled:

* AnswerDistributionWorkflow, enabled by `ANALYTICS_SCHEDULE_ANSWER_DISTRIBUTION`
* ImportEnrollmentsIntoMysql, enabled by `ANALYTICS_SCHEDULE_IMPORT_ENROLLMENTS_INTO_MYSQL`
* CourseActivityWeeklyTask, enabled by `ANALYTICS_SCHEDULE_COURSE_ACTIVITY_WEEKLY`
* InsertToMysqlAllVideoTask, enabled by `ANALYTICS_SCHEDULE_INSERT_TO_MYSQL_ALL_VIDEO`
* InsertToMysqlCourseEnrollByCountryWorkflow, enabled by `ANALYTICS_SCHEDULE_INSERT_TO_MYSQL_COURSE_ENROLL_BY_COUNTRY`

Each seed task clones the job-dsl repo, and an optional secure repo.

**Note:** There are two ways to specify a ssl-based github repo URL.  Note the
subtle difference in the paths: `github.com:your-org` vs. `github.com/your-org`.

* git@github.com:your-org/private-repo.git ✓ 
* ssh://git@github.com/your-org/private-repo.git ✓ 

*Not like this:*

* git@github.com/your-org/private-repo.git ❌ 
* ssh://git@github.com:your-org/private-repo.git ❌ 

##### Job DSL repo

The Job DSL repo is defined by these variables.

  `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_URL`: "git@github.com:edx-ops/edx-jenkins-job-dsl.git"
  `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_VERSION`: "master"
  `ANALYTICS_SCHEDULE_JOBS_DSL_REPO_CREDENTIAL_ID`: "{{ `JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID` }}"

The Job DSL repo is expected to contain:

  * `jobs/analytics-edx-jenkins.edx.org/*Jobs.groovy`: the seed job DSL
  * `jobs/analytics-edx-jenkins.edx.org/analyticstasks/*.groovy`: the analytics job DSLs

##### Secure repo

Analytics jobs require configuration that must be kept secure, e.g., ssh
credentials, EMR details.  Because this configuration must often be passed to
the EMR instances themselves, it's convenient to store it in a secure repo, and
share the repo between the seed tasks and the analytics tasks.

If you do not wish to store your analytics configuration in a repo, you can
configure it by setting the default `ANALYTICS_SCHEDULE_EXTRA_VARS`, or each
individual task's `ANALYTICS_SCHEDULE_<TASK_NAME>_EXTRA_VARS`.

If you do wish to store your analytics configuration in a repo, set the
following variables:

* `ANALYTICS_SCHEDULE_SECURE_REPO_URL`: `null`
* `ANALYTICS_SCHEDULE_SECURE_REPO_DEST`: "analytics-secure-config"
* `ANALYTICS_SCHEDULE_SECURE_REPO_VERSION`: "master"
* `ANALYTICS_SCHEDULE_SECURE_REPO_CREDENTIAL_ID`: "{{ `JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID` }}"

And then set `ANALYTICS_SCHEDULE_EXTRA_VARS`, or each individual task's
`ANALYTICS_SCHEDULE_<TASK_NAME>_EXTRA_VARS` to point to a file in the secure
repo.  For example:

        ANALYTICS_SCHEDULE_SECURE_REPO_URL: 'git@github.com:open-craft/analytics-secure-configuration.git'
        ANALYTICS_SCHEDULE_SECURE_REPO_VERSION: 'client-branch'
        ANALYTICS_SCHEDULE_EXTRA_VARS: "{{ ANALYTICS_SCHEDULE_SECURE_REPO_DEST }}/analytics-pipeline-vars.yml'
        ANALYTICS_SCHEDULE_ANSWER_DISTRIBUTION_EXTRA_VARS: "{{ ANALYTICS_SCHEDULE_SECURE_REPO_DEST }}/analytics-pipeline-answer-dist-vars.yml'

##### Analytics task configuration

To set a single configuration for all analytics tasks, use
`ANALYTICS_SCHEDULE_EXTRA_VARS`.  To specify different configuration for an
individual task, use `ANALYTICS_SCHEDULE_<TASK_NAME>_EXTRA_VARS`.

The variables to set depend on the job DSLs, which will provide sane defaults
for as many parameters as possible.  Consult the job DSL README for more
information.

#### Jenkins credentials

Jenkins contains its own credential store. To fill it with credentials, 
please use the `JENKINS_ANALYTICS_CREDENTIALS` variable. This variable 
is a list of objects, each object representing a single credential.
For now passwords and ssh-keys are supported. 

If you only need credentials to access github repositories
you can override `JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID`,
which should contain contents of private key used for 
authentication to checkout github repositories.  

Each credential has a unique ID, which is used to match 
the credential to the task(s) for which it is needed

Examples of credentials variables:
 
    JENKINS_ANALYTICS_GITHUB_CREDENTIAL_ID: "{{ lookup('file', 'path to keyfile') }}" 
        
    JENKINS_ANALYTICS_CREDENTIALS:
      # id is a scope-unique credential identifier
      - id: test-password
        # Scope must be global. To have other scopes you'll need to modify addCredentials.groovy
        scope: GLOBAL
        # Username associated with this password
        username: jenkins
        type: username-password
        description: Autogenerated by ansible
        password: 'password'
      # id is a scope-unique credential identifier
      - id: github-deploy-key
        scope: GLOBAL
        # Username this ssh-key is attached to
        username: git
        # Type of credential, see other entries for example
        type: ssh-private-key        
        passphrase: 'foobar'
        description: Generated by ansible
        privatekey: |
          -----BEGIN RSA PRIVATE KEY-----
          Proc-Type: 4,ENCRYPTED
          DEK-Info: AES-128-CBC,....

          Key contents
          -----END RSA PRIVATE KEY-----

#### Other useful variables

* `JENKINS_ANALYTICS_CONCURRENT_JOBS_COUNT`: Configures number of 
  executors (or concurrent jobs this Jenkins instance can 
  execute). Defaults to `2`. 

### General configuration 

Following variables are used by this role:

Variables used by command waiting on Jenkins start-up after running
`jenkins_master` role:

    jenkins_connection_retries: 60
    jenkins_connection_delay: 0.5

#### Auth realm

Jenkins auth realm encapsulates user management in Jenkins, that is:

* What users can log in
* What credentials they use to log in

Realm type stored in `jenkins_auth_realm.name` variable.

In future we will try to enable other auth domains, while
preserving the ability to run cli.

##### Unix Realm

For now only `unix` realm supported -- which requires every Jenkins
user to have a shell account on the server.

Unix realm requires the following settings:

* `service`: Jenkins uses PAM configuration for this service. `su` is
a safe choice as it doesn't require a user to have the ability to login
remotely.
* `plain_password`:  plaintext password, **you should change** default values.
* `hashed_password`: hashed password

Example realm configuration:

    jenkins_auth_realm:
      name: unix
      service: su
      plain_password: jenkins
      hashed_password: $6$rAVyI.p2wXVDKk5w$y0G1MQehmHtvaPgdtbrnvAsBqYQ99g939vxrdLXtPQCh/e7GJVwbnqIKZpve8EcMLTtq.7sZwTBYV9Tdjgf1k.


Known issues
------------

1. Playbook named `execute_ansible_cli.yaml`, should be converted to an
   Ansible module (it is already used in a module-ish way).
2. Anonymous user has discover and get job permission, as without it
   `get-job`, `build <<job>>` commands wouldn't work.
   Giving anonymous these permissions is a workaround for
   transient Jenkins issue (reported [couple][1] [of][2] [times][3]).
3. We force unix authentication method -- that is, every user that can login
   to Jenkins also needs to have a shell account on master.


Dependencies
------------

- `jenkins_master`

[1]: https://issues.jenkins-ci.org/browse/JENKINS-12543
[2]: https://issues.jenkins-ci.org/browse/JENKINS-11024
[3]: https://issues.jenkins-ci.org/browse/JENKINS-22143
