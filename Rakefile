require 'tempfile'
require 'rubygems'
require 'date'

desc 'Deploy'
task :deploy => [:clean, :compress] do
  sh "bin/deploy"
end

desc 'Compile scss, Compress generated css'
task :css do
  sh "mkdir -p css"
  scss = FileList['scss/**/*.scss'].exclude('scss/**/_*.scss')
  scss.each do |source|
    target = source.sub(/\.scss$/, ".css").sub(/^scss/, 'css')
    sh "sass -t compressed --cache-location /tmp #{source} #{target}"
  end
end

desc "Delete generate files"
task :clean do
  sh "rm css -rf"
  sh "rm _site -rf"
end

desc "Generate site"
task :generate => :css do
  sh "jekyll --no-auto"
end

desc "Watch for change"
task :watch => :generate do
  sh "while inotifywait -r -e modify scss/ _layouts _posts _includes;
       do rake generate; done"
end

desc "Compress html"
task :compress => :generate do
  html_srcs = FileList['_site/**/*.html']
  html_srcs.each do |h|
    sh "java -jar bin/htmlcompressor-1.3.1.jar --charset utf8 #{h} -o #{h}"
  end
end


# sudo gem install jekyll
# sudo apt-get install python-pygments
# sudo gem install liquid --version '2.2.2'
