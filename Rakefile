require "bundler/setup"
require "rubygems"
require "yaml"


task :default => [:build]

task :build do
  install_vagrant_plugins
  install_gems
end

def install_vagrant_plugins
	plugins = YAML.load_file('vagrant_plugins.yml')
	#puts plugins.inspect
	command = plugins.map { |plugin| "vagrant plugin install #{plugin['name']} --plugin-version #{plugin['version']}"}.join(" && ")

  Bundler.with_clean_env do
    fail "vagrant plugin installation failed" unless system(command)
  end
end


def install_vagrant_plugins
  Bundler.with_clean_env do
    command = "\
    	 vagrant plugin install vagrant-omnibus --plugin-version 1.1.0 \
      && vagrant plugin install vagrant-cachier --plugin-version 0.2.0 \
      && vagrant plugin install vagrant-plugin-bundler --plugin-version 0.1.1 \
      && vagrant plugin install vagrant-berkshelf --plugin-version 1.3.3 \
      && vagrant plugin install vagrant-vbguest --plugin-version 0.8.0"
    fail "vagrant plugin installation failed" unless system(command)
  end
end

def install_gems
  Bundler.with_clean_env do
    # XXX: DOH! - with_clean_env does not clear GEM_HOME if the rake task is invoked using `bundle exec`, 
    # which results in gems being installed to your current Ruby's GEM_HOME rather than Bills Kitchen's GEM_HOME!!! 
    fail "must run `rake build` instead of `bundle exec rake build`" if ENV['GEM_HOME']
    command = "gem install bundler -v 1.3.5 --no-ri --no-rdoc && bundle install --gemfile=Gemfile --verbose"
    fail "gem installation failed" unless system(command)
  end
end