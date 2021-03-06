# frozen_string_literal: true

require 'rubygems'
require 'bundler/setup'

require 'puppetlabs_spec_helper/rake_tasks'
require 'puppet/version'
require 'puppet-lint/tasks/puppet-lint'
require 'puppet-syntax/tasks/puppet-syntax'
require 'metadata-json-lint/rake_task'
require 'rubocop/rake_task'
require 'puppet-strings'
require 'puppet-strings/tasks'
require_relative 'spec/spec_utilities'
require 'nokogiri'
require 'open-uri'

def v(ver) ; Gem::Version.new(ver) ; end

if v(Puppet.version) >= v('4.9')
  require 'semantic_puppet'
elsif v(Puppet.version) >= v('3.6') && v(Puppet.version) < v('4.9')
  require 'puppet/vendor/semantic/lib/semantic'
end

# These gems aren't always present, for instance
# on Travis with --without development
begin
  require 'puppet_blacksmith/rake_tasks'
rescue LoadError # rubocop:disable Lint/HandleExceptions
end

RuboCop::RakeTask.new

exclude_paths = [
  'coverage/**/*',
  'doc/**/*',
  'pkg/**/*',
  'vendor/**/*',
  'spec/**/*'
]

Rake::Task[:lint].clear

PuppetLint.configuration.relative = true
PuppetLint.configuration.disable_80chars
PuppetLint.configuration.disable_class_inherits_from_params_class
PuppetLint.configuration.disable_class_parameter_defaults
PuppetLint.configuration.fail_on_warnings = true

PuppetLint::RakeTask.new :lint do |config|
  config.ignore_paths = exclude_paths
end

PuppetSyntax.exclude_paths = exclude_paths

task :beaker => :spec_prep

desc 'Run all non-acceptance rspec tests.'
RSpec::Core::RakeTask.new(:spec_unit) do |t|
  t.pattern = 'spec/{classes,templates,unit}/**/*_spec.rb'
end
task :spec_unit => :spec_prep

desc 'Run syntax, lint, and spec tests.'
task :test => %i[
  lint
  rubocop
  validate
  spec_unit
]

desc 'remove outdated module fixtures'
task :spec_prune do
  mods = 'spec/fixtures/modules'
  fixtures = YAML.load_file '.fixtures.yml'
  fixtures['fixtures']['forge_modules'].each do |mod, params|
    next unless params.is_a? Hash \
      and params.key? 'ref' \
      and File.exist? "#{mods}/#{mod}"

    metadata = JSON.parse(File.read("#{mods}/#{mod}/metadata.json"))
    FileUtils.rm_rf "#{mods}/#{mod}" unless metadata['version'] == params['ref']
  end
end
task :spec_prep => [:spec_prune]

# Plumbing for snapshot tests
desc 'Run the snapshot tests'
RSpec::Core::RakeTask.new("beaker:snapshot") do |task|
  task.rspec_opts = ['--color']
  task.pattern = 'spec/acceptance/tests/snapshot.rb'

  if Rake::Task.task_defined? 'artifact:snapshot:not_found'
    puts 'No snapshot artifacts found, skipping snapshot tests.'
    exit(0)
  end
end

beaker_node_sets.each do |node|
  desc "Run the snapshot tests against the #{node} nodeset"
  task "beaker:#{node}:snapshot" => %w[
    spec_prep
    artifact:snapshot:deb
    artifact:snapshot:rpm
  ] do
    ENV['BEAKER_set'] = node
    Rake::Task['beaker:snapshot'].reenable
    Rake::Task['beaker:snapshot'].invoke
  end
end

namespace :artifact do
  namespace :snapshot do
    dls = Nokogiri::HTML(open('https://www.elastic.co/downloads/kibana'))
    div = dls.at_css('#preview-release-id')

    if div.nil?
      puts 'No preview release available; skipping snapshot download'
      %w[deb rpm].each { |ext| task ext }
      task 'not_found'
    else
      div
        .at_css('.downloads')
        .xpath('li/a[contains(text(), "RPM") or contains(text(), "DEB")]')
        .each do |anchor|
          filename = artifact(anchor.attr('href'))
          link = artifact("kibana-snapshot.#{anchor.text.split(' ').first.downcase}")
          task anchor.text.split(' ').first.downcase => link
          file link => filename do
            unless File.exist?(link) and File.symlink?(link) \
                and File.readlink(link) == filename
              File.delete link if File.exist? link
              File.symlink File.basename(filename), link
            end
          end
          file filename do
            get anchor.attr('href'), filename
          end
      end
    end
  end
end
