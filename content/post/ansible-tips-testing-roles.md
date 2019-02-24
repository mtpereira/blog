---
title: "Ansible Tips: Testing roles using Vagrant and TravisCI"
aliases:
 - "ansible-tips-testing-roles.html"
 - "ansible-tips-testing-roles"
date: 2015-06-02T00:26:46Z
tags: ["ansible", "testing", "vagrant", "travisci", "ci", "ghost"]
draft: false
---

It's very useful to keep your roles on [TravisCI](https://travis-ci.org) or other publicly available CI system: it enables you to verify that your commits and other contributions to your role don't break it.

On the other hand, it's a little tedious to be constantly waiting for the CI to finish and give you some results. Heck, you might even feel tempted to `git push` without actually checking if your changes are OK! ;)

For that reason, I find it really useful to keep a set of tools on each role's repository to be able to check if I'm breaking anything whilst introducing new features or modifying existing ones.

I'll be using the role [mtpereira.ghost](https://github.com/mtpereira/ansible-ghost) as an example.

# Playbook for testing

A `test.yml` playbook is used for both TravisCI and Vagrant in order to run the role and testing the resulting host's behaviour on both environments.

A first play is used for **setting up** our tests:

{{< highlight yaml >}}
    - name: tests - apply role
      hosts: all
      tags: ghost_tests_apply
    
      roles:
        - .
    
      post_tasks:
	    - name: tests - sleep for Ghost startup
	      pause:
	        seconds: 15
	        prompt: "Waiting for Ghost startup..."
    
        - name: tests - install curl
	        apt:
	          pkg: curl
	          install_recommends: no
	          state: present
	        sudo: yes
{{< / highlight >}}

Notice that in order to be able to execute the role from the current directory, it's necessary to reference it by its path, `.`. This has several advantages over including all the role's files (tasks, handlers, vars, etc) directly onto the playbook, which is something that I've done previously on several roles:

- You're actually running the role as a whole, which might prove very different than running its individual components. This is **crucial** when you're role has dependencies on other roles, as they'll only be executed when the role is invoked. In this example, the role depends on external roles and, as such, it's necessary to guarantee that they're executed properly and that they provide the required behaviour.
- When you'll add new files to the role, you won't have to remember to add them to the `test.yml` playbook.
- It's cleaner and easier to read.

Also notice the `post_tasks` block after the role. These are for ensuring that we're in conditions to actually run the tests and are very specific to this case.

The next play includes both the **execution** and **validation** steps of the tests:

{{< highlight yaml >}}
	- name: tests - run tests
	  hosts: all
	  tags: ghost_tests_run
    
	  tasks:
	    - name: tests - check if ghost is listening
	      command: curl --silent {{ ghost_config_url }}
	      changed_when: false
	      ignore_errors: yes
	      register: ghost_test_listening
    
	    - name: tests - check why ghost is not listening
	      command: node {{ ghost_install_dir }}/index.js
	      args:
	        chdir: "{{ ghost_install_dir }}"
	        env:
	          NODE_ENV: production
	      changed_when: false
	      ignore_errors: yes
	      sudo_user: "{{ ghost_user_name }}"
	      sudo: yes
	      when: ghost_test_listening | failed
    
	    - name: tests - assert that Ghost is listening
	      assert:
	        that: "{{ ghost_test_listening | success }}"
    
	    - name: tests - check if Nginx is proxying Ghost
	      command: curl --silent localhost:{{ ghost_nginx_port }}
	      changed_when: false
	      ignore_errors: yes
	      register: ghost_test_nginx_proxy
    
	    - name: tests - assert that Nginx is proxying Ghost
	      assert:
	        that: "{{ ghost_test_nginx_proxy | success }}"
    
	    - name: tests - check if Ghost admin is protected from outside
	      command: curl --silent --head localhost:{{ ghost_nginx_port }}/ghost/setup/
	      changed_when: false
	      ignore_errors: yes
	      register: ghost_test_nginx_admin
    
	    - name: tests - assert that Ghost admin is protected from outside
	      assert:
	        that: "'404 Not Found' in ghost_test_nginx_admin.stdout" 
{{< / highlight >}}

There are a few note-worthy details here:

- `ignore_errors: yes` ensures that even if our exercise setup returns an erroneous exit code, we'll still get to assert it, which effectively separates the execution and the validation on different tasks.
- `changed_when: false` avoids that our tests change the idempotence tests' results on TravisCI.
- Testing using Ansible is not that pretty but is simple enough for testing role's as a atomic component of larger playbooks (which should probably be tested using something like [Serverspec](http://serverspec.org/).

# Ansible Galaxy requirements file

If the role depends on other roles, it's necessary to add a `requirements.yml` file for fetching said roles both locally and on TravisCI:

{{< highlight yaml >}}
	- src: nodesource.node
	  version: master
    
	- src: jdauphant.nginx
	  version: v1.3.4
{{< / highlight >}}


# Vagrantfile

Usually I keep a Vagrantfile that looks something like this.

{{< highlight ruby >}}
	boxes = {
	  "ghost-debian-wheezy" => {
	                        :box => "opscode-debian-7.8",
	                        :url => "https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_debian-7.8_chef-provisionerless.box",
	                        :ip  => '10.0.21.2',
	                        :cpu => "100",
	                        :ram => "128"
	                      },
	  "ghost-ubuntu-trusty" => {
	                        :box => "opscode-ubuntu-14.04",
	                        :url => "https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_ubuntu-14.04_chef-provisionerless.box",
	                        :ip  => '10.0.21.3',
	                        :cpu => "100",
	                        :ram => "128"
	                      },
	}
    
	Vagrant.configure("2") do |config|
	  boxes.each do |box_name, box|
	    config.vm.define box_name do |machine|
	      machine.vm.box = box[:box]
	      machine.vm.box_url = box[:url]
	      machine.vm.hostname = "%s" % box_name

	      machine.vm.provider "virtualbox" do |v|
	        v.customize ["modifyvm", :id, "--cpuexecutioncap", box[:cpu]]
	        v.customize ["modifyvm", :id, "--memory",          box[:ram]]
	      end
    
	      machine.vm.network :private_network, ip: box[:ip]
    
	      machine.vm.provision :shell do |shell|
	        shell.inline = "sed -i -e 's/%sudo\tALL=NOPASSWD:ALL/%sudo\tALL=(ALL:ALL) ALL/' /etc/sudoers"
	      end
    
	      machine.vm.provision :ansible do |ansible|
	        ansible.playbook = "test.yml"
	        ansible.verbose = ENV['ANSIBLE_VERBOSE'] ||= "v"
	        ansible.tags = ENV['ANSIBLE_TAGS'] ||= "all"
	      end
	    end
	  end
	end
{{< / highlight >}}

A multihost Vagrantfile allows for testing against multiple operating systems, which might prove difficult to do on a public CI system. In this case, we have a Ubuntu and a Debian host. If we run `vagrant up` we'll provision both VMs but we can also specify a name, so that it'll only provision that host:

{{< highlight bash >}}
vagrant up ghost-ubuntu-trusty
{{< / highlight >}}

The `ANSIBLE_VERBOSE` environment variable enables controlling Ansible's verbosity whilst still using Vagrant for testing:

{{< highlight bash >}}
ANSIBLE_VERBOSE=vv vagrant provision
{{< / highlight >}}

The same goes for `ANSIBLE_TAGS`, for controlling which tags to run. This is very useful for decreasing the write-test-debug cycle on specific Ansible tasks. Tagging your role properly is crucial for this, which we'll explore on a future post.

# TravisCI

The `.travis.yml` file that I use here is the same found on most roles: it tests the role and then tests it's idempotence. It uses the same exact `test.yml` playbook used on Vagrant. The only note-worthy detail here is that we need to run `ansible-galaxy` on the `before_script` block:

{{< highlight yaml >}}
install:
  - pip install ansible==1.9.1
  - "printf '[defaults]\\nroles_path = ../' > ansible.cfg"

before_script:
  - echo localhost > inventory
  - ansible-galaxy install --role-file requirements.yml
{{< / highlight >}}

# Conclusion

With these tools, I find that the role's development cycle usually goes something like this:

1. Provision hosts and run test playbook with `vagrant up`
2. Fix bugs
3. Re-run the affected tasks using `ANSIBLE_TAGS=tag_name vagrant provision`
4. Repeat 2. and 3. as needed
5. Reprovision host for full role test: `vagrant destroy && vagrant up`
6. `git push` and check CI output for errors
7. Go to 1. if necessary
8. `vagrant destroy` for clean up

If you have any question or if you've found some mistake on this post, please drop me a line using [any of my contacts](about).

# Resources

1. ["Testing Ansible Roles with Travis CI on GitHub"](https://servercheck.in/blog/testing-ansible-roles-travis-ci-github)
