TARGET_VERSION = '1.9.2-p136'.freeze

TMP_DIR = File.expand_path('./destroot').freeze
TARBALL = "ruby-#{TARGET_VERSION}.tar.gz".freeze
PACKAGES = %w[ ruby rubydoc rubytk rubydev goruby ].freeze
PROTOTYPES = PACKAGES.map {|pkg| "prototype.#{pkg}" }.freeze

autoload :ERB, 'erb'

def open_files(*specs, &block)
  ios = []
  begin
    specs.each do |args|
      ios << File.open(*args)
    end
    block.call(*ios)
  ensure
    ios.each do |io|
      io.close rescue nil
    end
  end
end

def editable_files
  @editable_file ||= (
    miniruby = File.join("ruby-#{TARGET_VERSION}", "miniruby")
    platform = `#{miniruby} -e 'RUBY_PLATFORM.display'`

    %W[ 
      bin/rdoc bin/gem bin/irb bin/testrb bin/rake bin/ri bin/erb
      include/ruby-1.9.1/#{platform}/ruby/config.h
      lib/ruby/1.9.1/#{platform}/rbconfig.rb
    ]
  )
end

def src_dir
  "ruby-#{TARGET_VERSION}"
end

task :default => :package

task :package => PROTOTYPES + %w[ postinstall make_relocatable ] do
  pwd = Dir.pwd
  PACKAGES.each do |pkg|
    sh "pkgmk -f prototype.#{pkg} -o -r #{TMP_DIR}/opt/local -a `uname -p` -d #{pwd}/pkg"
    sh "pkgtrans -s #{pwd}/pkg #{pwd}/pkg/YUGUI#{pkg}.pkg YUGUI#{pkg}"
  end
end

task :make_relocatable => 'build.timestamp' do
  editable_files.each do |path|
    path = File.join(TMP_DIR, 'opt', 'local', path)
    contents = File.read(path).gsub('/opt/local', '##PREFIX##')
    File.open(path, "w"){|f| f.write contents }
  end
end

file 'postinstall' => ['postinstall.erb', 'Rakefile'] do
  erb = ERB.new(File.read("postinstall.erb"))
  File.open("postinstall", "w"){|f| f.write erb.result }
end

PROTOTYPES.each do |name|
  file name => "build.timestamp" do
    Rake::Task[:prototype].invoke
  end
end

task :prototype do
  specs = PROTOTYPES.map{|name| [name, "w"] }
  open_files(*specs){|ruby, rubydoc, rubytk, rubydev, goruby|
    ruby.write <<-EOS.gsub(/^\s+/, '')
      i pkginfo=./pkginfo.ruby
      i depend=./depend.ruby
      i copyright=ruby-#{TARGET_VERSION}/COPYING
      i postinstall=./postinstall
    EOS

    [ [rubydoc, "rubydoc"], [rubytk, "rubytk"], [rubydev, "rubydev"], [goruby, "goruby"] ].each do |out, suffix|
      out.write <<-EOS.gsub(/^\s+/, '')
        i pkginfo=./pkginfo.#{suffix}
        i depend=./depend.#{suffix}
        i copyright=ruby-#{TARGET_VERSION}/COPYING
      EOS
    end

    pat = %r[#{TMP_DIR}/opt/local/]o
    IO.popen("find #{TMP_DIR}/opt/local | pkgproto", "r"){|pipe|
      pipe.each do |line|
        if reloc_path = line[/^[bcdefilpsvx] \w+ (.+?) [0-7]{4} \w+ \w+$/, 1]
          case reloc_path
          when TMP_DIR, "#{TMP_DIR}/opt", "#{TMP_DIR}/opt/local"
            next
          when /\s/ # work around for 1.9.2-p0; rdoc generates files whose paths contain SPs.
            next
          end

          line[pat] = reloc_path[pat] = ""

          out = case reloc_path
                when %r[^share/ri] then 
                  rubydoc
                when %r[^lib/ruby/1\.9\.1/tk], %r[/tcltklib\.so$], %r[/tkutil\.so$] then
                  rubytk
                when %r[^include/ruby], %r[^lib/ruby/1\.9\.1/mkmf\.rb], %r[^lib/libruby-static\.a] then
                  rubydev
                when %r[^bin/goruby], %r[^share/man/man1/goruby\.1] then 
                  goruby
                else 
                  ruby
                end

          line[/\w+ \w+$/] = "root root"
          if editable_files.include?(reloc_path)
            out.write line.sub(/^f/, 'e')
          else
            out.write line
          end
        else
          line[pat] = ""
          ruby.write line
        end
      end
    }
  }
end

file 'build.timestamp' => ['Rakefile', TARBALL] do
  Rake::Task[:build].invoke
  touch 'build.timestamp'
end

task :clean do
  rm_rf TMP_DIR
  rm_rf src_dir
end

task :build => [TARBALL, :clean] do
  rm_rf src_dir
  sh "gzip -dc #{TARBALL} | tar xf -"
  Dir.chdir(src_dir){
    sh "./configure --prefix=/opt/local CC=cc MAKE=make --with-opt-dir=/opt/csw --enable-shared"
    sh "make"
    sh "make golf"
    sh "make install DESTDIR=#{TMP_DIR}"
  }
end

file TARBALL do
  sh "curl 'http://ftp.ruby-lang.org/pub/ruby/1.9/#{TARBALL}' > #{TARBALL}"
end
