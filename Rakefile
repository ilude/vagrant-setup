%w{ bundler/setup rubygems fileutils uri net/https tmpdir digest/md5 yaml win32/registry Win32API }.each do |file|
  require file
end


BASE_DIR = File.expand_path('.', File.dirname(__FILE__)) 
#TARGET_DIR  = "#{BASE_DIR}/target" 
BUILD_DIR   = BASE_DIR # "#{BASE_DIR}/target/build"
CACHE_DIR   = "#{BASE_DIR}/cache"
#ZIP_EXE = 'C:\Program Files\7-Zip\7z.exe'
HWND_BROADCAST = 0xffff
WM_SETTINGCHANGE = 0x001A
SMTO_ABORTIFHUNG = 2

task :default => [:build]

task :build do
  create_cache
  install_vagrant
  install_vagrant_plugins
  install_gems
end

def create_cache
  FileUtils.mkdir_p CACHE_DIR
end

def install_vagrant_plugins
  plugins = YAML.load_file('vagrant_plugins.yml')

  command = ["#{BUILD_DIR}/set-env.bat"]
  command << plugins.map { |plugin| "vagrant plugin install #{plugin['name']} --plugin-version #{plugin['version']}"}
  command = command.join(" && ")

  Bundler.with_clean_env do
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

def install_vagrant
  download_and_unpack "http://files.vagrantup.com/packages/0219bb87725aac28a97c0e924c310cc97831fd9d/Vagrant_1.2.4.msi", "#{BUILD_DIR}/", ''

  #add vagrant to users path
  vagrant_bin_dir = File.join(BUILD_DIR, "HashiCorp/Vagrant/embedded/bin").gsub('/', '\\')
  env = Win32::Registry::HKEY_CURRENT_USER.open('Environment', Win32::Registry::KEY_ALL_ACCESS)
  env['Path'] = (env['Path'].split(';') << vagrant_bin_dir).join(';') unless env['Path'].include? vagrant_bin_dir

  sendMessageTimeout = Win32API.new('user32', 'SendMessageTimeout', 'LLLPLLP', 'L') 
  result = 0
  sendMessageTimeout.call(HWND_BROADCAST, WM_SETTINGCHANGE, 0, 'Environment', SMTO_ABORTIFHUNG, 5000, result)
end

def download_and_unpack(url, target_dir, includes = []) 
  Dir.mktmpdir do |tmp_dir| 
    outfile = "#{tmp_dir}/#{File.basename(url)}"
    download(url, outfile)
    unpack(outfile, target_dir, includes)
  end
end

def download(url, outfile)
  puts "checking cache for '#{url}'"
  url_hash = Digest::MD5.hexdigest(url)
  cached_file = "#{CACHE_DIR}/#{url_hash}"
  if File.exist? cached_file
    puts "cache-hit: read from '#{url_hash}'"
    FileUtils.cp cached_file, outfile
  else
    download_no_cache(url, outfile)
    puts "caching as '#{url_hash}'"
    FileUtils.cp outfile, cached_file
  end
end

def download_no_cache(url, outfile, limit=5)

  raise ArgumentError, 'HTTP redirect too deep' if limit == 0

  puts "download '#{url}'"
  uri = URI.parse url
  if ENV['HTTP_PROXY']
    proxy_host, proxy_port = ENV['HTTP_PROXY'].sub(/https?:\/\//, '').split ':'
    puts "using proxy #{proxy_host}:#{proxy_port}"
    http = Net::HTTP::Proxy(proxy_host, proxy_port.to_i).new uri.host, uri.port
  else
    http = Net::HTTP.new uri.host, uri.port
  end

  if uri.port == 443
    http.verify_mode = OpenSSL::SSL::VERIFY_NONE
    http.use_ssl = true
  end 

  http.start do |agent|
    agent.request_get(uri.path + (uri.query ? "?#{uri.query}" : '')) do |response|
      # handle 301/302 redirects
      redirect_url = response['location']
      if(redirect_url)
        puts "redirecting to #{redirect_url}"
        download_no_cache(redirect_url, outfile, limit - 1)
      else
        File.open(outfile, 'wb') do |f|
          response.read_body do |segment|
            f.write(segment)
          end
        end
      end
    end
  end
end

def unpack(archive, target_dir, includes = [])
  case File.extname(archive)
  when '.zip', '.7z', '.exe'
    puts "extracting '#{archive}' to '#{target_dir}'"
    system("\"#{ZIP_EXE}\" x -o\"#{target_dir}\" -y \"#{archive}\" -r #{includes.join(' ')} 1> NUL")
  when '.msi'
    puts "installing '#{archive}' to '#{target_dir}'"
    system("start /wait msiexec /a \"#{archive.gsub('/', '\\')}\" /qb TARGETDIR=\"#{target_dir.gsub('/', '\\')}\"")
    FileUtils.rm_r File.join(target_dir, File.basename(archive))
  else 
    raise "don't know how to unpack '#{archive}'"
  end
end

