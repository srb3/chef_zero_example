require 'fileutils'
require 'net/ssh'
require 'chef_zero/server'
require 'yaml'

def workspace
  prop('workspace')
end

def dirs
  %W(
    data
    .chef
    #{workspace}/cookbooks
    #{workspace}/environments
    #{workspace}/roles
    #{workspace}/nodes
    #{workspace}/data_bags
  )
end  

def prop(key)
  YAML.load_file('config.yml')[key]
end

namespace :setup do

  desc 'create all'
  task 'create' do
    Rake::Task['setup:create_dirs'].invoke
    Rake::Task['setup:create_pem'].invoke
    Rake::Task['setup:create_knife'].invoke
    Rake::Task['setup:gen_data'].invoke
  end

  desc 'create the chef repo'
  task :create_dirs do
    dirs.each do |d|
      FileUtils.mkdir_p d
    end
  end

  desc 'create pem file'
  task :create_pem do
    key = OpenSSL::PKey::RSA.new 2048
    IO.write('.chef/client.pem',key.to_pem) 
  end

  desc 'create knife.rb file'
  task :create_knife do
    str = <<~EOF
      chef_repo         = File.join(File.dirname(__FILE__), '..')
      chef_server_url   'http://#{prop('host')}:#{prop('port')}'
      node_name         'dummynode'
      client_key        File.join(File.dirname(__FILE__), 'client.pem')
      cookbook_path     "\#{chef_repo}/cookbooks"
      cache_type        'BasicFile'
      cache_options     :path => "\#{chef_repo}/checksums"
    EOF
    IO.write('.chef/knife.rb',str) 
  end

  desc 'generate chef server data'
  task :gen_data do
    require 'net/http'
    require 'json'
    require 'uri'

    uri = URI.parse('https://raw.githubusercontent.com/dwyl/english-words/master/words_dictionary.json')
    response = JSON.parse(Net::HTTP.get_response(uri).body).keys.shuffle
    prop('data_bags').each do |k,v|
      p = "#{workspace}/data_bags/#{k}"
      next if Dir[File.join(p, '**', '*')].count { |file| File.file?(file) } >= v
      FileUtils.mkdir_p p
      (1..v.to_i).each do |i|
        x = <<~EOF
        {
         "id": "#{response[i]}",
         "some_data": "#{(0...8).map { (65 + rand(26)).chr }.join}"
        }
        EOF
        IO.write("#{p}/#{response[i]}.json",x)
      end
    end
    response.shuffle!
    (1..prop('nodes')).each do |n|
      p = "#{workspace}/nodes/"
      next if Dir[File.join(p, '**', '*')].count { |file| File.file?(file) } >= prop('nodes')
      (1..prop('nodes').to_i).each do |i|
        x = <<~EOF
        {
          "name": "#{response[i]}",
          "chef_type": "node",
          "json_class": "Chef::Node",
          "chef_environment": "staging",
          "run_list": []
        }
        EOF
        IO.write("#{workspace}/nodes/#{response[i]}.json",x)
      end
    end
    response.shuffle!
    (1..prop('cookbooks')).each do |n|
      p = "#{workspace}/cookbooks/"
      next if Dir[File.join(p, '**', '*')].count { |dir| File.directory?(dir) } >= prop('cookbooks')
      (1..prop('cookbooks').to_i).each do |i|
        FileUtils.mkdir_p "#{p}/#{response[i]}/recipes"
        metadata = <<~EOF
        name "#{response[i]}"
        version "1.0.0"
        EOF
        recipe = <<~EOF
        package '#{response[i]}'
        EOF
        IO.write("#{p}/#{response[i]}/metadata.rb",metadata)
        IO.write("#{p}/#{response[i]}/recipes/default.rb",recipe)
      end
    end
   response.shuffle!
    (1..prop('environments')).each do |n|
      p = "#{workspace}/environments/"
      next if Dir[File.join(p, '**', '*')].count { |file| File.file?(file) } >= prop('environments')
      (1..prop('environments').to_i).each do |i|
        x = <<~EOF
        {
          "name": "#{response[i]}",
          "description": "This is just #{response[i]}",
          "json_class": "Chef::Environment",
          "chef_type": "environment",
          "cookbook_versions": {
          },
          "default_attributes": {},
          "override_attributes": {}
        }

        EOF
        IO.write("#{p}/#{response[i]}.json",x)
      end
    end
  end
end

namespace :destroy do
  desc 'remove the chef repo'
  task :delete_chef_repo do
    dirs.each do |d|
      FileUtils.rm_rf d
    end
  end

  desc 'remove the log file'
  task :delete_log_file do
    FileUtils.rm_f 'chef-server.log'
  end

  desc 'clean all'
  task 'clean' do
    Rake::Task['destroy:delete_chef_repo'].invoke
    Rake::Task['destroy:delete_log_file'].invoke
  end

end
