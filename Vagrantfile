# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

# initialize a few values
options = {}
VALID_ZK_ENSEMBLE_SIZES = [3, 5, 7]

# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate we are provisioning a cluster (if multiple
# nodes are supplied via the `--zookeeper-list` flag)
provisioning_command_args = ['up', 'provision']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:zookeeper_list] = nil
  opts.on( '-z', '--zookeeper-list A1,A2[,...]', 'Zookeeper address list (multi-node commands)' ) do |zookeeper_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-z=192.168.1.1')
    options[:zookeeper_list] = zookeeper_list.gsub(/^=/,'')
  end

  options[:zookeeper_path] = nil
  opts.on( '-p', '--path ZK_DIR', 'Zookeeper installation path' ) do |zookeeper_path|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-p=/opt/zookeeper')
    options[:zookeeper_path] = zookeeper_path.gsub(/^=/,'')
  end

  options[:zookeeper_url] = nil
  opts.on( '-u', '--url ZK_URL', 'URL for distribution' ) do |zookeeper_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:zookeeper_url] = zookeeper_url.gsub(/^=/,'')
  end

  options[:local_zk_file] = nil
  opts.on( '-l', '--local-zk-file File', 'Local file containing Zookeeper distribution' ) do |local_zk_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=/tmp/tmp.tgz')
    options[:local_zk_file] = local_zk_file.gsub(/^=/,'')
  end

  options[:yum_repo_url] = nil
  opts.on( '-y', '--yum-url URL', 'Local yum repository URL' ) do |yum_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-y=http://192.168.1.128/centos')
    options[:yum_repo_url] = yum_repo_url.gsub(/^=/,'')
  end

  options[:zookeeper_data_dir] = nil
  opts.on( '-d', '--data DATA_DIR', 'Data directory for Zookeeper' ) do |zookeeper_data_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-r="/data"')
    options[:zookeeper_data_dir] = zookeeper_data_dir.gsub(/^=/,'')
  end

  options[:local_vars_file] = nil
  opts.on( '-f', '--local-vars-file FILE', 'Local variables file' ) do |local_vars_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-f=/tmp/local-vars-file.yml')
    options[:local_vars_file] = local_vars_file.gsub(/^=/,'')
  end

  options[:reset_proxy_settings] = false
  opts.on( '-c', '--clear-proxy-settings', 'Clear existing proxy settings if no proxy is set' ) do |reset_proxy_settings|
    options[:reset_proxy_settings] = true
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

# parse the command-line options that were passed in
begin
  optparse.order_recognized!(ARGV)
rescue SystemExit => e
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)

# if a local variables file was passed in, check and make sure it's a valid filename
if options[:local_vars_file] && !File.file?(options[:local_vars_file])
  print "ERROR; input local variables file '#{options[:local_vars_file]}' is not a local file\n"
  exit 3
end

# if a Zookeeeper URL was passed in, check to make sure it looks like a URI
if options[:zookeeper_url] && !(options[:zookeeper_url] =~ URI::regexp)
  print "ERROR; input zookeeper URL '#{options[:zookeeper_url]}' is not a valid URL\n"
  exit 3
end

# if a local Zookeeper file was passed in, check to make sure it's a valid filename
if options[:local_zk_file] && !File.exists?(options[:local_zk_file])
  print "ERROR; input local zookeeper file '#{options[:local_zk_file]}' is not a local file\n"
  exit 3
end

zookeeper_addr_array = []
# if we're provisioning or an IP address is (or addresses are) required for this command,
# then either the `--node` flag should be provided and only contain a single node or the
# `--zookeeper-list` flag should be used to pass in the addresses for multiple nodes that
# define a valid definition for zookeeper ensemble we're deploying
if provisioning_command || ip_required
  if !options[:zookeeper_list]
    print "ERROR; IP address must be supplied (using the `-z, --zookeeper-list` flag) for this vagrant command\n"
    exit 1
  else
    zookeeper_addr_array = options[:zookeeper_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    if single_ip_command && zookeeper_addr_array.size > 1
      print "ERROR; Only a single IP address can be supplied (using the `-z, --zookeeper-list` flag) for this vagrant command\n"
      exit 2
    elsif zookeeper_addr_array.size == 1
      if !(zookeeper_addr_array[0] =~ Resolv::IPv4::Regex)
        print "ERROR; input zookeeper IP address #{zookeeper_addr_array[0]} is not a valid IP address\n"
        exit 2
      end
    elsif !single_ip_command
      # when provisioning a multi-node Zookeeper ensemble, we **must** have a zookeeper cluster
      # consisting of an odd number of nodes greater than three, but less than seven (any other
      # topology is not supported, so an error is thrown)
      not_ip_addr_list = zookeeper_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input zookeeper IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input zookeeper IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      elsif !(VALID_ZK_ENSEMBLE_SIZES.include?(zookeeper_addr_array.size))
        print "ERROR; only a zookeeper cluster with an odd number of elements between three and\n"
        print "       seven is supported for multi-node zookeeper deployments; requested cluster\n"
        print "       #{zookeeper_addr_array} contains #{zookeeper_addr_array.size} elements\n"
        exit 5
      end
    end
  end
end

# if a yum repository address was passed in, check and make sure it's a valid URL
if options[:yum_repo_url] && !(options[:yum_repo_url] =~ URI::regexp)
  print "ERROR; input yum repository URL '#{options[:yum_repo_url]}' is not a valid URL\n"
  exit 6
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if zookeeper_addr_array.size > 0
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if $proxy
        config.proxy.http               = $proxy
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if $no_proxy
        config.proxy.no_proxy           = $no_proxy
      end
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    config.vm.box_check_update = false
    # loop through all of the addresses in the `zookeeper_addr_array` and, if we're
    # creating VMs, create a VM for each machine; if we're just provisioning the
    # VMs using an ansible playbook, then wait until the last VM in the loop and
    # trigger the playbook runs for all of the nodes simultaneously using the
    # `site.yml` playbook
    zookeeper_addr_array.each do |machine_addr|
      config.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
      end
      config.vm.define machine_addr do |machine|
        # setup a private network for this machine
        machine.vm.network "private_network", ip: machine_addr
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes sumultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == zookeeper_addr_array[-1]
          # use the playbook in the `site.yml' file to provision Zookeeper to our nodes
          machine.vm.provision "ansible" do |ansible|
            # set the limit to 'all' in order to provision all of machines on the
            # list in a single playbook run
            ansible.limit = "all"
            ansible.playbook = "site.yml"
            ansible.groups = {
              zookeeper: zookeeper_addr_array
            }
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: proxy,
                no_proxy: no_proxy,
                proxy_username: proxy_username,
                proxy_password: proxy_password
              },
              zookeeper_iface: "eth1",
              yum_repo_url: options[:yum_repo_url],
              local_zk_file: options[:local_zk_file],
              host_inventory: zookeeper_addr_array,
              reset_proxy_settings: options[:reset_proxy_settings],
              cloud: "vagrant"
            }
            # if a Zookeeper data directory was set, then set an extra variable
            # containing the named directory
            if options[:zookeeper_data_dir]
              ansible.extra_vars[:zookeeper_data_dir] = options[:zookeeper_data_dir]
            end
            # set the `zookeeper_url` and `zookeeper_path` extra variables, if values for these
            # parameters were included on the command-line
            if options[:zookeeper_url]
              ansible.extra_vars[:zookeeper_url] = options[:zookeeper_url]
            end
            # if defined, set the 'extra_vars[:zookeeper_dir]' value to the value that was passed in on
            # the command-line (eg. "/opt/zookeeper")
            if options[:zookeeper_path]
              ansible.extra_vars[:zookeeper_dir] = options[:zookeeper_path]
            end
            # if defined, set the 'extra_vars[:local_vars_file]' value to the value that was passed in
            # on the command-line (eg. "/tmp/local-vars-file.yml")
            if options[:local_vars_file]
              ansible.extra_vars[:local_vars_file] = options[:local_vars_file]
            end
          end     # end `machine.vm.provision "ansible" do |ansible|`
        end     # end `if machine_addr == zookeeper_addr_array[-1]`
      end     # end `config.vm.define machine_addr do |machine|`
    end     # end `zookeeper_addr_array.each do |machine_addr|`
  end     # end `Vagrant.configure ("2") do |config|`
end     # end `if zookeeper_addr_array.size > 0`
