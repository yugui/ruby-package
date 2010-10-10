TMP_DIR = File.expand_path('./destroot')
TARGET_VERSION = '1.9.2-p0'
TARBALL = "ruby-#{TARGET_VERSION}.tar.gz"
autoload :ERB, 'erb'

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

task :default => :package

task :package => %w[ prototype postinstall make_relocatable ] do
  pwd = Dir.pwd
  %w[ ruby rubydoc rubytk ].each do |pkg|
    sh "pkgmk -f prototype.#{pkg} -o -r #{TMP_DIR}/opt/local -a `uname -p` -d #{pwd}/pkg"
    sh "pkgtrans -s #{pwd}/pkg #{pwd}/pkg/YUGUI#{pkg}.pkg YUGUI#{pkg}"
  end
end

task :make_relocatable do
  editable_files.each do |path|
    path = File.join(TMP_DIR, 'opt', 'local', path)
    contents = File.read(path).gsub('/opt/local', '##PREFIX##')
    File.open(path, "w"){|f| f.write contents }
  end
end

task :postinstall do
  erb = ERB.new(File.read("postinstall.erb"))
  File.open("postinstall", "w"){|f| f.write erb.result }
end

#task :prototype => :build do
task :prototype do
  File.open("prototype.ruby", "w"){|ruby|
    ruby.write <<-EOS.gsub(/^\s+/, '')
      i pkginfo=./pkginfo.ruby
      i depend=./depend.ruby
      i copyright=ruby-#{TARGET_VERSION}/COPYING
      i postinstall=./postinstall
    EOS

    File.open("prototype.rubydoc", "w"){|rubydoc|
      rubydoc.write <<-EOS.gsub(/^\s+/, '')
        i pkginfo=./pkginfo.rubydoc
        i depend=./depend.rubydoc
        i copyright=ruby-#{TARGET_VERSION}/COPYING
      EOS

      File.open("prototype.rubytk", "w"){|rubytk|
        rubytk.write <<-EOS.gsub(/^\s+/, '')
          i pkginfo=./pkginfo.rubytk
          i depend=./depend.rubytk
          i copyright=ruby-#{TARGET_VERSION}/COPYING
        EOS

        IO.popen("find #{TMP_DIR}/opt/local | pkgproto", "r"){|pipe|
          pipe.each do |line|
            next if /d none #{TMP_DIR} [0-7]{4} \w+ \w+/ =~ line
            next if /d none #{TMP_DIR}\/opt [0-7]{4} \w+ \w+/ =~ line
            next if /d none #{TMP_DIR}\/opt\/local [0-7]{4} \w+ \w+/ =~ line
    
            # work around for 1.9.2-p0
            next if %r[share/ri/1.9.1/system/IRB/\(MagicFile = ] =~ line
            next if %r[share/ri/1.9.1/system/Set/dig = ] =~ line
    
            line = line.sub("#{TMP_DIR}/opt/local/", '')
            line = line.sub(/([0-7]{4}) \w+ \w+/, '\1 root root')

            out = case line
                  when %r[share/ri] then rubydoc
                  when %r[lib/ruby/1.9.1/tk] then rubytk
                  when %r[tcltklib.so] then rubytk
                  when %r[tkutil.so] then rubytk
                  else ruby
                  end
    
            if editable_files.any?{|path| line == "f none #{path} 0755 root root\n" }
              out.write line.sub(/^f/, 'e')
            else
              out.write line
            end
          end
        }
      }
    }
  }
end

task :build => TARBALL do
  rm_rf TMP_DIR
  dir = "ruby-#{TARGET_VERSION}"

  rm_rf dir
  sh "gzip -dc #{TARBALL} | tar xf -"
  Dir.chdir(dir){
    sh "./configure --prefix=/opt/local CC=cc MAKE=make --with-opt-dir=/opt/csw --enable-shared"
    sh "make"
    sh "make install DESTDIR=#{TMP_DIR}"
  }
end

file TARBALL do
  sh "curl 'http://ftp.ruby-lang.org/pub/ruby/1.9/#{TARBALL}' > #{TARBALL}"
end
