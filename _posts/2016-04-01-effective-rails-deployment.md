---
layout: post
title: Effective Rails Deployment
---

[Capistrano](http://capistranorb.com/) has been an excellent tool for years to deploy simple or multi-stage complex applications to remote servers. However the rise of PASS like Heroku, Dokku, I got in mind to use some automatic orchestration framework.

I looked over [Chef](https://www.chef.io/chef/), [Puppet](https://puppet.com/), [SaltStack](http://saltstack.com/) and [Ansible](https://www.ansible.com/) on a weekend and decided to go with ansible since it looked easiest to get hands dirty and have great documentation. After some initial hicups I was able to deploy some of our inhouse applications on EC2 and introduced it to my coworkers at [codecrux](http://codecrux.com).

Since then, we use ansible to configure and manage remote server, and also configure our local development environment running on top of Virtualbox with vagrant.

Ansible uses YAML for configuration and for actual commands it has less flexibility compared to capistrano which uses Ruby DSL, but at the same time helps you to keep your playbooks simple. ansible also extremely good with orchestration and rolling deployments. We were able to keep a homogenous command set and duplicate most of Capistrano’s features in very small amount of code. 

Let's take an example to deploy [lobster.rs](https://github.com/jcs/lobsters). Here is how we did it - 

**Configuration**

First of all you need to define required variables in a yaml file for ansible to pick them up.


    ---
    app_name: lobsters
    rails_env: production

    git_url: git@github.com:jcs/lobsters.git
    git_version: master

    app_path: '/{{ app_name }}'
    shared_path: '{{ app_path }}/shared'
    releases_path: '{{ app_path }}/releases'
    current_release_path: '{{ app_path }}/current'
    app_public_path: "{{ current_release_path }}/public"
    app_config_path: "{{ current_release_path }}/config"
    app_temp_path: "{{ current_release_path }}/tmp"
    app_logs_path: "{{ current_release_path }}/log"

    keep_releases: 5

You should also define your remote host in ansible inventory/host file like

    [production]
    lobsters.dev   ## replace with your domain name

**Playbook**

Now when we have configuration we can create the actual playbook for doing capistrano-style deployments. Create `deploy.yml` file:

    ---
    - hosts: all
      tasks:
        - set_fact: this_release_ts={{ lookup('pipe', 'date +%Y%m%d%H%M%S') }}
        - set_fact: this_release_path={{ releases_path }}/{{ this_release_ts }}

        - debug: msg='New release path {{ this_release_path }}'

        - name: Create new release dir
          file: path={{ this_release_path }} state=directory

        - name: Update code
          git: repo={{ git_url }} dest={{ this_release_path }} version={{ git_version }} accept_hostkey=yes
          register: git

        - debug: msg='Updated repo from {{ git.before }} to {{ git.after }}'

        - name: Symlink shared files
          file: src={{ shared_path }}/{{ item }} dest={{ this_release_path }}/{{ item }} state=link force=yes
          with_items:
            - config/database.yml
            - config/secrets.yml
            - config/unicorn.rb
            - log
            - tmp
            - vendor/bundle

        - name: Install bundle
          command: 'bundle install --deployment --without="development test"'
          args:
            chdir: '{{ this_release_path }}'

        - name: Precompile assets
          command: rake assets:precompile chdir={{ this_release_path }}
          environment:
            RAILS_ENV: '{{ rails_env }}'

        - name: Migrate database
          command: rake db:migrate chdir={{ this_release_path }}
          environment:
            RAILS_ENV: '{{ rails_env }}'

        - name: Symlink new release
          file: src={{ this_release_path }} dest={{ current_release_path }} state=link force=yes

        - name: Restart unicorn
          command: sudo restart {{ app_name }}

        - name: Cleanup
          shell: "ls -1t {{ releases_path }}|tail -n +{{ keep_releases + 1 }}|xargs rm -rf"
          args:
            chdir: '{{ releases_path }}'


**Deploy**

Once everything is set up, you can run the following command to deploy:

    ansible-playbook -u railsbox -i production deploy.yml


