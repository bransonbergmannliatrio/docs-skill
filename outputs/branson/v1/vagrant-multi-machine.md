# Vagrant Multi-Machine

**Source:** <https://developer.hashicorp.com/vagrant/docs/multi-machine>

**TL;DR** — A single `Vagrantfile` can define and control several guest machines. Define each with `config.vm.define`; each definition block gets a scoped config object that overlays the shared `config`. Once more than one machine exists, single-target commands like `vagrant ssh` require a machine name, while `vagrant up` acts on all machines by default (and accepts a name or a `/regex/`).

**At a glance**
- Define machines — `config.vm.define "name" do |m| … end`
- Control machines — name-targeted vs. all-machine commands; regex matching
- Communicate between machines — use the networking options
- Primary machine — `primary: true` sets the default target
- Autostart — `autostart: false` keeps a machine out of `vagrant up`

## Defining multiple machines

Define machines in one project `Vagrantfile` with `config.vm.define`. The block variable (e.g. `web`) is the same as `config` except its configuration applies only to that machine.

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision "shell", inline: "echo Hello"

  config.vm.define "web" do |web|
    web.vm.box = "apache"
  end

  config.vm.define "db" do |db|
    db.vm.box = "mysql"
  end
end
```

You can still use the outer `config` object; it is loaded and merged before machine-specific configuration, following the [Vagrantfile load order](https://developer.hashicorp.com/vagrant/docs/vagrantfile#load-order). This is like variable scoping in a programming language.

**Ordering is outside-in**, in the order written in the file. For the Vagrantfile below, provisioners output `A`, then `C`, then `B` — the machine-scoped `B` runs last:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision :shell, inline: "echo A"

  config.vm.define :testing do |test|
    test.vm.provision :shell, inline: "echo B"
  end

  config.vm.provision :shell, inline: "echo C"
end
```

To vary configuration across many similar machines, see the [loop-over-definitions tip](https://developer.hashicorp.com/vagrant/docs/vagrantfile/tips#loop-over-vm-definitions).

## Controlling multiple machines

Once more than one machine is defined, `vagrant` command behavior changes:

- **Single-target commands** (e.g. `vagrant ssh`) require a machine name: `vagrant ssh web` or `vagrant ssh db`.
- **All-machine commands** (e.g. `vagrant up`) act on every machine by default, or on a named one: `vagrant up web`.
- **Regex matching** — a name inside forward slashes is treated as a regular expression. `vagrant up /follower[0-9]/` brings up `follower0`, `follower1`, … but not `leader`.

## Communication between machines

Use the [networking](https://developer.hashicorp.com/vagrant/docs/networking) options. A [private network](https://developer.hashicorp.com/vagrant/docs/networking/private_network) creates a private network among the machines and the host.

## Specifying a primary machine

The primary machine is the default target when no machine is specified. Mark it when defining it; only one primary may be set.

```ruby
config.vm.define "web", primary: true do |web|
  # ...
end
```

## Autostart machines

`vagrant up` starts all defined machines by default. Set `autostart: false` to exclude a machine; start it later by naming it explicitly.

```ruby
config.vm.define "web"
config.vm.define "db"
config.vm.define "db_follower", autostart: false
```

Here `vagrant up` starts `web` and `db` but not `db_follower`; run `vagrant up db_follower` to start it manually.
