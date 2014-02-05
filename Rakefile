
require "pathname"
require 'rake/clean'

CLEAN.include("publisher.pdf","publisher.log","publisher.protocol","publisher.vars")
CLOBBER.include("build/sourcedoc","src/go/sp/sp","src/go/sp/docgo", "src/go/sp/bin","src/go/sp/pkg")

installdir = Pathname.new(__FILE__).join("..")
srcdir   = installdir.join("src")
builddir = installdir.join("build")
@versions = {}
File.read("version").each_line do |line|
	product,versionnumber = line.chomp.split(/=/) # / <-- ignore this slash
	@versions[product]=versionnumber
end



desc "Compile and install necessary software"
task :build do
	ENV['GOPATH'] = "#{srcdir}/go/sp"
	publisher_version = @versions['publisher_version']
	Dir.chdir(srcdir.join("go","sp")) do
		puts "Building (and copying) sp binary..."
  		sh "go build -ldflags \"-X main.dest git -X main.version #{publisher_version}\" -o  #{installdir}/bin/sp  main"
  		puts "...done"
	end
end

desc "Generate documentation"
task :doc do
	Dir.chdir(installdir.join("doc","manual")) do
		sh "jekyll build"
	end
	print "Now generating command reference from XML..."
	mkdir_p "temp"
	sh "java -Dfile.encoding=utf8 -jar #{installdir}/lib/saxon9he.jar -s:#{installdir}/doc/commands-xml/commands.xml -o:/dev/null -xsl:#{installdir}/doc/commands-xml/xslt/cmd2html.xsl lang=en builddir=#{builddir}/manual 2> temp/messages-en.csv"
	sh "java -Dfile.encoding=utf8 -jar #{installdir}/lib/saxon9he.jar -s:#{installdir}/doc/commands-xml/commands.xml -o:/dev/null -xsl:#{installdir}/doc/commands-xml/xslt/cmd2html.xsl lang=de builddir=#{builddir}/manual 2> temp/messages-de.csv"
	puts "done"
end

desc "Generate schema and translations from master"
task :schema do
  # generate the lua translation
  sh "java -jar #{installdir}/lib/saxon9he.jar -s:#{installdir}/schema/translations.xml -o:#{installdir}/src/lua/translations.lua -xsl:#{installdir}/schema/genluatranslations.xsl"
  # generate english + german schema
  sh "java -jar #{installdir}/lib/saxon9he.jar -s:#{installdir}/schema/layoutschema-master.rng -o:#{installdir}/schema/layoutschema-en.rng -xsl:#{installdir}/schema/translate_schema.xsl pFrom=en pTo=en"
  sh "java -jar #{installdir}/lib/saxon9he.jar -s:#{installdir}/schema/layoutschema-master.rng -o:#{installdir}/schema/layoutschema-de.rng -xsl:#{installdir}/schema/translate_schema.xsl pFrom=en pTo=de"
end

desc "Source documentation"
task :sourcedoc do
	Dir.chdir("#{srcdir}/lua") do
		sh "#{installdir}/third/locco/locco.lua #{builddir}/sourcedoc *lua publisher/*lua common/*lua fonts/*.lua barcodes/*lua"
	end
	ENV['GOPATH'] = "#{srcdir}/go/sp"
	Dir.chdir(srcdir.join("go","sp")) do
		puts "Building docgo..."
  		sh 'go build github.com/pgundlach/docgo'
  		puts "...done"
  		sh "./docgo -outdir #{builddir}/sourcedoc -resdir #{srcdir}/go/sp/src/github.com/pgundlach/docgo/ sp.go"
	end
	puts "done"
	puts "Generated source documentation in \n#{builddir}/sourcedoc"
end

desc "Update program messages"
task :messages do
	lang = "de_DE"
	Dir.chdir(srcdir) do
		srcfiles = Dir.glob("lua/**/*.lua")
		# xgettext creates the pot file
		sh 'xgettext --from-code="UTF-8" -k"log" -k"err" -k"warning" -s -o po/publisher.pot ' + srcfiles.join(" ")
		# msgmerge moves new messages to the po file
		sh "msgmerge -s -U po/#{lang}.po po/publisher.pot"
		# msgfmt creates the mo file
		sh "msgfmt -c -v -o po/#{lang}.mo po/#{lang}.po"
	end
end

desc "New language for program messages"
task :newmsglang, :lang do |t,args|
	unless args[:lang]
		raise "No language given. Use rake newmsglang[de_DE] to create a new language template."
	end
	Dir.chdir(srcdir) do
		lang = args[:lang]
		srcfiles = Dir.glob("lua/**/*.lua")
		sh 'xgettext --from-code="UTF-8" -k"log" -k"err" -k"warning" -s -o po/publisher.pot ' + srcfiles.join(" ")
		sh "msginit -l #{lang} -o po/#{lang}.po -i po/publisher.pot"
	end
end

desc "Update gh-pages"
task :ghpages => [:doc] do
	cp_r "#{builddir}/manual","webpage"
	sh "bin/create-dash-documentsets.py"
	Dir.chdir(builddir) do
		sh "tar --exclude='.DS_Store' -czf ../webpage/speedatapublisher-de.tgz speedatapublisher-de.docset"
		sh "tar --exclude='.DS_Store' -czf ../webpage/speedatapublisher-en.tgz speedatapublisher-en.docset"
	end
	IO.write("webpage/speedata_Publisher_(en).xml","<entry>\n  <version>#{@versions['publisher_version']}</version>\n  <url>http://speedata.github.io/publisher/speedatapublisher-en.tgz</url>\n</entry>\n")
	IO.write("webpage/speedata_Publisher_(de).xml","<entry>\n  <version>#{@versions['publisher_version']}</version>\n  <url>http://speedata.github.io/publisher/speedatapublisher-de.tgz</url>\n</entry>\n")
end

# For now: only a small test
desc "Test source code"
task :test do
	ENV["LUA_PATH"] = "#{srcdir}/lua/?.lua;#{installdir}/lib/?.lua;#{installdir}/test/?.lua"
	ENV["PUBLISHER_BASE_PATH"] = installdir.to_s
	inifile = srcdir.join("sdini.lua")
	sh "texlua --lua=#{inifile} #{installdir}/bin/luatest tc_xpath.lua"
end

desc "Run quality assurance"
task :qa do
	sh "#{installdir}/bin/sp compare #{installdir}/qa"
end

def build_go(srcdir,destbin,goos,goarch)
	ENV['GOARCH'] = goarch
	ENV['GOOS'] = goos
	ENV['GOPATH'] = "#{srcdir}/go/sp"
	publisher_version = @versions['publisher_version']

	binaryname = goos == "windows" ? "sp.exe" : "sp"
    # Now compile the go executable
	cmdline = "go build -ldflags '-X main.dest directory -X main.version #{publisher_version}' -o #{destbin}/#{binaryname} main"
	sh cmdline do |ok, res|
		if ! ok
	    	puts "Go compilation failed"
	    	return false
	    end
	end
	return true
end

desc "Make ZIP files - set NODOC=true for stripped zip file"
task :zip do
	srcbindir = ENV["LUATEX_BIN"] || ""
	if ! test(?d,srcbindir) then
		puts "Environment variable LUATEX_BIN does not exist.\nMake sure it points to a path which contains `luatex'.\nUse like this: rake zip LUATEX_BIN=/path/to/bin\nAborting"
		next
	end
	publisher_version = @versions['publisher_version']
	dest="#{builddir}/speedata-publisher"
	targetbin="#{dest}/bin"
	targetshare="#{dest}/share"
	targetfonts=File.join(targetshare,"fonts")
	targetschema=File.join(targetshare,"schema")
	targetsw=File.join(dest,"sw")
	rm_rf dest
	mkdir_p dest
	mkdir_p targetbin
	mkdir_p File.join(targetshare,"img")
	mkdir_p targetschema
	mkdir_p targetsw
	if ENV['NODOC'] != "true" then
		Rake::Task["doc"].execute
		cp_r "#{builddir}/manual",File.join(targetshare,"doc")
	end
	platform = nil
	arch = nil
	if test(?f, srcbindir +"/sdluatex") then
		cp_r(srcbindir +"/sdluatex",targetbin)
	elsif test(?f,srcbindir +"/luatex.exe") then
		cp_r(srcbindir +"/luatex.exe",targetbin)
		cp_r(srcbindir +"/luatex.dll",targetbin)
		cp_r(srcbindir +"/kpathsea610.dll",targetbin)
		platform = "windows"
		arch = "386"
	elsif test(?f,srcbindir +"/luatex") then
		cp_r(srcbindir +"/sdluatex","#{targetbin}/sdluatex")
	end
	if platform == nil then
		res = `file #{targetbin}/sdluatex`.gsub(/^.*sdluatex:/,'')
		case res
		when /Mach-O/
			platform = "darwin"
		when /Linux/
			platform = "linux"
		end
		case res
		when /x86-64/,/x86_64/,/64-bit/
			arch = "amd64"
		when /32-bit/
			arch = "386"
		end
	end
	if !platform or !arch then
		puts "Could not determine architecture (amd64/386) or platform (windows/linux/darwin)"
		puts "This is the output of 'file':"
		puts res
		next
	end

	zipname = "speedata-publisher-#{platform}-#{arch}-#{publisher_version}.zip"
	rm_rf File.join(builddir,zipname)

	if build_go(srcdir,targetbin,platform,arch) == false then next end
	cp_r(File.join("fonts"),targetfonts)
	cp_r(Dir.glob("img/*"),File.join(targetshare,"img"))
	cp_r(File.join("lib"),targetshare)
	cp_r(File.join("schema","layoutschema-de.rng"),targetschema)
	cp_r(File.join("schema","layoutschema-en.rng"),targetschema)

	Dir.chdir("src") do
		cp_r(["tex","hyphenation"],targetsw)
		# do not copy every Lua file to the dest
		# and leave out .gitignore and others
		Dir.glob("**/*.lua").reject { |x|
		    x =~  /viznode/
		}.each { |x|
		  FileUtils.mkdir_p(File.join(targetsw , File.dirname(x)))
		  FileUtils.cp(x,File.join(targetsw,File.dirname(x)))
		}
	end
	dirname = "speedata-publisher"
	cmdline = "zip -rq #{zipname} #{dirname}"
	Dir.chdir("build") do
		puts cmdline
		puts `#{cmdline}`
	end
end
