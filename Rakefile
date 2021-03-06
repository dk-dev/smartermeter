require 'rubygems'
require 'rake'
require 'date'

#############################################################################
#
# Helper functions
#
#############################################################################

def name
  @name ||= Dir['*.gemspec'].first.split('.').first
end

def version
  line = File.read("lib/#{name}.rb")[/^\s*VERSION\s*=\s*.*/]
  line.match(/.*VERSION\s*=\s*['"](.*)['"]/)[1]
end

def date
  Date.today.to_s
end

def rubyforge_project
  name
end

def gemspec_file
  "#{name}.gemspec"
end

def gem_file
  "#{name}-#{version}.gem"
end

def replace_header(head, header_name)
  head.sub!(/(\.#{header_name}\s*= ').*'/) { "#{$1}#{send(header_name)}'"}
end

#############################################################################
#
# Standard tasks
#
#############################################################################

task :default => :spec

require 'rspec/core/rake_task'
RSpec::Core::RakeTask.new(:spec)

desc "Generate code coverage"
RSpec::Core::RakeTask.new(:coverage) do |t|
  t.rcov = true
  t.rcov_opts = ["--exclude", "spec"]
end

require 'rdoc/task'
Rake::RDocTask.new do |rdoc|
  rdoc.rdoc_dir = 'rdoc'
  rdoc.title = "#{name} #{version}"
  rdoc.rdoc_files.include('README*')
  rdoc.rdoc_files.include('lib/**/*.rb')
end

desc "Open an irb session preloaded with this library"
task :console do
  sh "irb -rubygems -r ./lib/#{name}.rb"
end

#############################################################################
#
# Custom tasks (add your own tasks here)
#
#############################################################################

task :fetch_jruby do
  require 'net/http'
  require 'uri'

  dir = File.join(File.dirname(__FILE__), "pkg", "base")
  jar_file = File.join(dir, "jruby-complete.jar")
  FileUtils.mkdir_p(dir)
  unless File.exists?(jar_file)
    jruby_url = "http://jruby.org.s3.amazonaws.com/downloads/1.6.1/jruby-complete-1.6.1.jar"
    puts "Fetching JRuby"
    File.open(jar_file, "wb") { |f| f.write(Net::HTTP.get(URI.parse(jruby_url))) }
    puts "Fetched JRuby"
  end
end

task :fetch_launch4j do
  require 'net/http'
  require 'uri'
  require 'zlib'
  require 'bundler'
  Bundler.setup
  require 'archive/tar/minitar'

  dir = File.join(File.dirname(__FILE__), "installer")
  tgz_file = File.join(dir, "launch4j-3.0.2-linux.tgz")
  FileUtils.mkdir_p(dir)
  unless File.exists?(tgz_file)
    launch4j_url = "http://softlayer.dl.sourceforge.net/project/launch4j/launch4j-3/3.0.2/launch4j-3.0.2-linux.tgz"
    puts "Fetching Launch4J"
    File.open(tgz_file, "wb") { |f| f.write(Net::HTTP.get(URI.parse(launch4j_url))) }
    puts "Fetched Launch4J"
    puts "Extracting Launch4J"
    tgz = Zlib::GzipReader.new(File.open(tgz_file, 'rb'))
    Archive::Tar::Minitar.unpack(tgz, dir)
    puts "Extracted Launch4J"
  end
end

task :package_gems do
  specifications = Dir.glob(File.join(File.dirname(__FILE__), "vendor", "gems", "jruby", "1.8", "specifications", "*"))
  specifications.reject! { |s| s.include? "rspec" }
  specifications.reject! { |s| s.include? "minitar" }
  specifications.reject! { |s| s.include? "ruby-debug" }
  specifications.reject! { |s| s.include? "addressable" }
  specifications.reject! { |s| s.include? "diff-lcs" }
  specifications.reject! { |s| s.include? "vcr" }
  specifications.reject! { |s| s.include? "webmock" }
  dest_dir = File.join(File.dirname(__FILE__), "pkg", "base", "gems", "specifications")
  FileUtils.rm_rf(dest_dir)
  FileUtils.mkdir_p(dest_dir)
  FileUtils.cp_r(specifications, dest_dir)

  gems = Dir.glob(File.join(File.dirname(__FILE__), "vendor", "gems", "jruby", "1.8", "gems", "*"))
  gems.reject! { |s| s.include? "rspec" }
  gems.reject! { |s| s.include? "minitar" }
  gems.reject! { |s| s.include? "ruby-debug" }
  gems.reject! { |s| s.include? "addressable" }
  gems.reject! { |s| s.include? "diff-lcs" }
  gems.reject! { |s| s.include? "vcr" }
  gems.reject! { |s| s.include? "webmock" }
  dest_dir = File.join(File.dirname(__FILE__), "pkg", "base", "gems", "gems")
  FileUtils.rm_rf(dest_dir)
  FileUtils.mkdir_p(dest_dir)
  FileUtils.cp_r(gems, dest_dir)
end

task :create_launcher do
  configuration = File.join(File.dirname(__FILE__), "installer", "launch4j.xml")
  `installer/launch4j/launch4j #{configuration}`
end

task :copy_smartermeter do
  src_dir = File.join(File.dirname(__FILE__), "lib")
  dest_dir = File.join(File.dirname(__FILE__), "pkg", "base", "gems", "gems", "smartermeter-#{version}")
  FileUtils.mkdir_p(dest_dir)
  FileUtils.cp_r(src_dir, dest_dir)

  dest = File.join(File.dirname(__FILE__), "pkg", "base", "gems", "specifications", "smartermeter-#{version}.gemspec")
  FileUtils.cp(File.join(File.dirname(__FILE__), "smartermeter.gemspec"), dest)

  dest_dir = File.join(File.dirname(__FILE__), "pkg", "base")
  FileUtils.cp(File.join(File.dirname(__FILE__), "installer", "main.rb"), dest_dir)

  dest_dir = File.join(File.dirname(__FILE__), "pkg", "base", "icons")
  FileUtils.mkdir_p(dest_dir)
  FileUtils.cp(Dir.glob(File.join(File.dirname(__FILE__), "icons", "*.png")), dest_dir)
end

task :nsis_installer do
  nsis_file = File.join(File.dirname(__FILE__), "installer", "nsis.nsi")
  `makensis -DVERSION=#{version} #{nsis_file}`
end

desc "Package all required files into pkg/base"
task :package => [:build, :fetch_jruby, :fetch_launch4j, :package_gems, :create_launcher, :copy_smartermeter, :nsis_installer]

#############################################################################
#
# Packaging tasks
#
#############################################################################

desc "Create tag v#{version} and build and push #{gem_file} to Rubygems"
task :release => :build do
  unless `git branch` =~ /^\* master$/
    puts "You must be on the master branch to release!"
    exit!
  end
  sh "git commit --allow-empty -a -m 'Release #{version}'"
  sh "git tag v#{version}"
  sh "git push origin master"
  sh "git push origin v#{version}"
  sh "gem push pkg/#{name}-#{version}.gem"
end

desc "Build #{gem_file} into the pkg directory"
task :build => :gemspec do
  sh "mkdir -p pkg"
  sh "gem build #{gemspec_file}"
  sh "mv #{gem_file} pkg"
end

desc "Generate #{gemspec_file}"
task :gemspec => :validate do
  # read spec file and split out manifest section
  spec = File.read(gemspec_file)
  head, manifest, tail = spec.split("  # = MANIFEST =\n")

  # replace name version and date
  replace_header(head, :name)
  replace_header(head, :version)
  replace_header(head, :date)
  #comment this out if your rubyforge_project has a different name
  replace_header(head, :rubyforge_project)

  # determine file list from git ls-files
  files = `git ls-files`.
    split("\n").
    sort.
    reject { |file| file =~ /^\./ }.
    reject { |file| file =~ /^(rdoc|pkg)/ }.
    reject { |file| file =~ /\.jar$/ }.
    reject { |file| file =~ /^spec\// }.
    map { |file| "    #{file}" }.
    join("\n")

  # piece file back together and write
  manifest = "  s.files = %w[\n#{files}\n  ]\n"
  spec = [head, manifest, tail].join("  # = MANIFEST =\n")
  File.open(gemspec_file, 'w') { |io| io.write(spec) }
  puts "Updated #{gemspec_file}"
end

desc "Validate #{gemspec_file}"
task :validate do
  libfiles = Dir['lib/*'] - ["lib/#{name}.rb", "lib/#{name}"]
  unless libfiles.empty?
    puts "Directory `lib` should only contain a `#{name}.rb` file and `#{name}` dir."
    exit!
  end
  unless Dir['VERSION*'].empty?
    puts "A `VERSION` file at root level violates Gem best practices."
    exit!
  end
end
