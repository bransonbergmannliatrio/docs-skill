# Vagrant Multi-Machine

**Sources:**
- `fixtures/vagrant-multi-machine.mdx` (primary fixture)
- <https://developer.hashicorp.com/vagrant/docs/multi-machine> (rendered primary)
- <https://developer.hashicorp.com/vagrant/docs/vagrantfile> (Vagrantfile load order & merging)
- <https://developer.hashicorp.com/vagrant/docs/vagrantfile/tips> (loop-over-VM-definitions tip)
- <https://developer.hashicorp.com/vagrant/docs/networking> (networking overview)
- <https://developer.hashicorp.com/vagrant/docs/networking/private_network> (private network)

> Source note: the fixture links to `/vagrant/docs/networking/private_network` (underscore). The hyphenated variant `/networking/private-network` returns **HTTP 404**; the underscore URL above resolves and is the one used here.

**TL;DR** — A single `Vagrantfile` can define and control several guest machines ("multi-machine"). Define each with `config.vm.define`; each definition block gets a scoped config object that overlays the shared outer `config`, merged in Vagrantfile load order. Once more than one machine exists, single-target commands like `vagrant ssh` require a machine name, while `vagrant up` acts on all machines by default (and accepts a name or a `/regex/`). Machines talk to each other over Vagrant's networking options — most commonly a **private network** — and you can mark one machine `primary` (default target) or set `autostart: false` to keep a machine out of `vagrant up`.

**At a glance**
- **Why multi-machine** — model multi-server topologies, distributed systems, interface testing, disaster-case testing.
- **Defining machines** — `config.vm.define "name" do |m| … end`; scoped vs. shared config.
- **Load & merge order** — where the outer `config` and machine overrides sit in the Vagrantfile load sequence.
- **Provisioner ordering** — outside-in, in file order.
- **Loop over definitions** — generate many similar machines (use `.each`, not `for`).
- **Controlling machines** — name-targeted vs. all-machine commands; regex matching.
- **Communication between machines** — networking options; private network for inter-machine + host connectivity.
- **Primary machine** — `primary: true` sets the default target.
- **Autostart** — `autostart: false` keeps a machine out of `vagrant up`.

## Why use multi-machine

Vagrant can define and control multiple guest machines per Vagrantfile. These machines generally work together or are associated with each other. Common use-cases:

- Accurately modeling a multi-server production topology, such as separating a web and database server.
- Modeling a distributed system and how the parts interact.
- Testing an interface, such as an API to a service component.
- Disaster-case testing: machines dying, network partitions, slow networks, inconsistent world views, etc.

Historically these environments were flattened onto a single machine, which is an inaccurate model of production and can behave far differently. Multi-machine models them within one Vagrant environment without losing the benefits of Vagrant.

## Defining multiple machines

Define machines in one project [`Vagrantfile`](https://developer.hashicorp.com/vagrant/docs/vagrantfile) with `config.vm.define`. This directive creates a Vagrant configuration within a configuration. The block variable (e.g. `web`) is the _exact_ same as the outer `config`, except any configuration on it applies **only** to the machine being defined. This is like variable scoping in a programming language.

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
*(fixture / multi-machine page)*

You can still use the outer `config` object as well; it is loaded and merged **before** the machine-specific configuration, just like other Vagrantfiles in the load order (see next section).

## Vagrantfile load order and merging

*(source: Vagrantfile page — provides the context for "loaded and merged before machine-specific configuration" that the fixture references)*

Vagrant loads Vagrantfiles in sequence, merging configuration progressively. Order (earlier is loaded first, later overrides):

1. **Box-packaged Vagrantfile** — configuration bundled with the box.
2. **User home directory** — `~/.vagrant.d/Vagrantfile`, for system-wide defaults.
3. **Project directory** — your primary `Vagrantfile` (most commonly edited).
4. **Multi-machine overrides** — machine-specific configuration (the `config.vm.define` block).
5. **Provider-specific overrides** — provider-dependent settings.

**Merging behavior:** settings set are merged with previous values. Most settings follow an **override** pattern — newer configuration replaces older. Some configurations (notably networking) **append** rather than override. Multiple `Vagrant.configure` blocks may appear in one Vagrantfile and merge sequentially in the order they appear.

**Lookup path:** on running a command, Vagrant searches upward from the current directory for the first Vagrantfile, up to the filesystem root. Override the search origin with the `VAGRANT_CWD` environment variable.

## Provisioner ordering

When using these scopes, order of execution for things like provisioners matters. Vagrant enforces ordering **outside-in**, in the order listed in the Vagrantfile.

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision :shell, inline: "echo A"

  config.vm.define :testing do |test|
    test.vm.provision :shell, inline: "echo B"
  end

  config.vm.provision :shell, inline: "echo C"
end
```
*(fixture / multi-machine page)*

The provisioners output **"A", then "C", then "B"**. "B" is last because ordering is outside-in, in the order of the file (the machine-scoped provisioner runs after the outer ones).

## Loop over VM definitions (many similar machines)

To apply a slightly different configuration to multiple machines, loop over the definitions. Use an iterator (`.each`), **not** a `for` loop.

```ruby
(1..3).each do |i|
  config.vm.define "node-#{i}" do |node|
    node.vm.provision "shell",
      inline: "echo hello from node #{i}"
  end
end
```
*(source: Vagrantfile tips page — "Loop over VM definitions")*

**Warning:** do not use `for i in 1..3` instead. Because multi-machine definitions are **lazy-loaded**, a `for` loop mutates a single `i` rather than creating an independent copy per iteration, so every node ends up provisioning with the same text (the value `i` holds after the loop). Vagrant cannot detect this automatically — it is an easy mistake to make.

## Controlling multiple machines

The moment more than one machine is defined, `vagrant` command behavior changes slightly (mostly intuitively):

- **Single-target commands** (e.g. `vagrant ssh`) now _require_ the machine name: `vagrant ssh web` or `vagrant ssh db`.
- **All-machine commands** (e.g. `vagrant up`) operate on _every_ machine by default (`vagrant up` brings up both `web` and `db`), or on a named one: `vagrant up web` / `vagrant up db`.
- **Regex matching** — a name inside forward slashes is treated as a regular expression. For a `leader` plus `follower0`, `follower1`, `follower2`, …, bring up all followers but not the leader with `vagrant up /follower[0-9]/`.

## Communication between machines

To let machines in a multi-machine setup communicate, use Vagrant's [networking](https://developer.hashicorp.com/vagrant/docs/networking) options.

Vagrant provides three high-level networking options (they are provider-independent abstractions, working across VirtualBox, VMware, etc.):

1. **Forwarded ports** — access a guest service from the host.
2. **Private network** — communication between host and guest, or **between multiple machines**.
3. **Public network** — connect the guest to a public network.

Vagrant assumes an available NAT device on `eth0`, ensuring it always has a way to communicate with the guest. Guidance: use only the high-level networking options until you are comfortable with the Vagrant workflow and have things working at a basic level, since provider-specific configuration can restrict access if misconfigured.

For multi-machine, the **[private network](https://developer.hashicorp.com/vagrant/docs/networking/private_network)** is the recommended way to make a private network between multiple machines and the host.

### Private network examples

*(source: networking/private_network page)*

A private network lets you reach the guest by an address not publicly accessible from the internet; multiple machines on the same private network can communicate with each other.

**DHCP** (automatic IP from the reserved space — discover it via `vagrant ssh` then `ifconfig` / `ip addr show`):

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", type: "dhcp"
end
```

**Static IP** (choose from reserved private address space to avoid conflicts / public routing):

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "192.168.50.4"
end
```

**IPv6** (static only — DHCP is unsupported; default netmask is 64):

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "fde4:8dba:82e1::c4"
end
```

**IPv6 with custom netmask:**

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network",
    ip: "fde4:8dba:82e1::c4",
    netmask: "96"
end
```

**Disable auto-configuration** (configure the interface manually with `auto_config: false`):

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "192.168.50.4",
    auto_config: false
end
```

## Specifying a primary machine

You can mark one machine as _primary_ — the default machine used when a specific machine is not named. Mark it when defining it; only one primary may be specified.

```ruby
config.vm.define "web", primary: true do |web|
  # ...
end
```
*(fixture / multi-machine page)*

## Autostart machines

By default, `vagrant up` starts all defined machines. Set `autostart: false` to tell Vagrant _not_ to start specific machines.

```ruby
config.vm.define "web"
config.vm.define "db"
config.vm.define "db_follower", autostart: false
```
*(fixture / multi-machine page)*

With these settings, `vagrant up` starts `web` and `db` but not `db_follower`. Force it to start by naming it: `vagrant up db_follower`.
