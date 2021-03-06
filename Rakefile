# Taken from http://ixti.net/software/2013/01/28/using-jekyll-plugins-on-github-pages.html
# necessary since some plugins needed to build site, and github-pages runs in safe
# mode (no arbitrary code execution).
require "tmpdir"

require "bundler/setup"
require "jekyll"
require "fileutils"

task :default => :generate

GITHUB_REPONAME = "singularperturbation/singularperturbation.github.io"

desc "Generate blog files"
task :generate do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => ".",
    "destination" => "_site"
  })).process
end

desc "Clean the generated files"
task :clean do
  Jekyll::Commands::Clean.process(
    "source"      => ".",
    "destination" => "_site",
  )
end

desc "Generate and publish blog to gh-pages"
task :publish => [:clean,:generate] do
  Dir.mktmpdir do |tmp|
    cp_r "_site/.", tmp

    pwd = Dir.pwd
    Dir.chdir tmp

    system "git init"
    system "git add ."
    message = "Site updated at #{Time.now.utc}"
    system "git commit -m #{message.inspect}"
    system "git remote add origin git@github.com:#{GITHUB_REPONAME}.git"
    system "git push origin master --force"

    Dir.chdir pwd
  end
end
