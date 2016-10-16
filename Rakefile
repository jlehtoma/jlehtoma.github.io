require "rubygems"
require 'rake'
require 'yaml'
require 'time'

require "bundler/setup"
require "jekyll"

SOURCE = "."
CONFIG = {
  'version' => "0.3.0",
  'themes' => File.join(SOURCE, "_includes", "themes"),
  'layouts' => File.join(SOURCE, "_layouts"),
  'posts' => File.join(SOURCE, "_posts"),
  'post_ext' => "md",
  'notes' => File.join(SOURCE, "literature/notes"),
  'lit_json' => File.join(SOURCE, "literature/literature.json"),
  'theme_package_version' => "0.1.0"
}

# Path configuration helper
module JB
  class Path
    SOURCE = "."
    Paths = {
      :layouts => "_layouts",
      :themes => "_includes/themes",
      :theme_assets => "assets/themes",
      :theme_packages => "_theme_packages",
      :posts => "_posts"
    }

    def self.base
      SOURCE
    end

    # build a path relative to configured path settings.
    def self.build(path, opts = {})
      opts[:root] ||= SOURCE
      path = "#{opts[:root]}/#{Paths[path.to_sym]}/#{opts[:node]}".split("/")
      path.compact!
      File.__send__ :join, path
    end

  end #Path
end #JB

# Usage: rake post title="A Title" [date="2012-02-09"] [tags=[tag1, tag2]]
desc "Begin a new post in #{CONFIG['posts']}"
task :post do
  abort("rake aborted: '#{CONFIG['posts']}' directory not found.") unless FileTest.directory?(CONFIG['posts'])
  title = ENV["title"] || "new-post"
  tags = ENV["tags"] || "[]"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  begin
    date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y-%m-%d')
  rescue => e
    puts "Error - date format must be YYYY-MM-DD, please check you typed it correctly!"
    exit -1
  end
  filename = File.join(CONFIG['posts'], "#{date}-#{slug}.#{CONFIG['post_ext']}")
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/,' ')}\""
    post.puts 'description: ""'
    post.puts "category: "
    post.puts "tags: #{tags}"
    post.puts "---"
    post.puts "{% include JB/setup %}"
  end
end # task :post

# Usage: rake page name="about.html"
# You can also specify a sub-directory path.
# If you don't specify a file extention we create an index.html at the path specified
desc "Create a new page."
task :page do
  name = ENV["name"] || "new-page.md"
  filename = File.join(SOURCE, "#{name}")
  filename = File.join(filename, "index.html") if File.extname(filename) == ""
  title = File.basename(filename, File.extname(filename)).gsub(/[\W\_]/, " ").gsub(/\b\w/){$&.upcase}
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end

  mkdir_p File.dirname(filename)
  puts "Creating new page: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: page"
    post.puts "title: \"#{title}\""
    post.puts 'description: ""'
    post.puts "---"
    post.puts "{% include JB/setup %}"
  end
end # task :page

desc "Launch preview environment"
task :preview do
  system "jekyll serve -w"
end # task :preview

# Public: Alias - Maintains backwards compatability for theme switching.
task :switch_theme => "theme:switch"

namespace :theme do

  # Public: Switch from one theme to another for your blog.
  #
  # name - String, Required. name of the theme you want to switch to.
  #        The theme must be installed into your JB framework.
  #
  # Examples
  #
  #   rake theme:switch name="the-program"
  #
  # Returns Success/failure messages.
  desc "Switch between Jekyll-bootstrap themes."
  task :switch do
    theme_name = ENV["name"].to_s
    theme_path = File.join(CONFIG['themes'], theme_name)
    settings_file = File.join(theme_path, "settings.yml")
    non_layout_files = ["settings.yml"]

    abort("rake aborted: name cannot be blank") if theme_name.empty?
    abort("rake aborted: '#{theme_path}' directory not found.") unless FileTest.directory?(theme_path)
    abort("rake aborted: '#{CONFIG['layouts']}' directory not found.") unless FileTest.directory?(CONFIG['layouts'])

    Dir.glob("#{theme_path}/*") do |filename|
      next if non_layout_files.include?(File.basename(filename).downcase)
      puts "Generating '#{theme_name}' layout: #{File.basename(filename)}"

      open(File.join(CONFIG['layouts'], File.basename(filename)), 'w') do |page|
        if File.basename(filename, ".html").downcase == "default"
          page.puts "---"
          page.puts File.read(settings_file) if File.exist?(settings_file)
          page.puts "---"
        else
          page.puts "---"
          page.puts "layout: default"
          page.puts "---"
        end
        page.puts "{% include JB/setup %}"
        page.puts "{% include themes/#{theme_name}/#{File.basename(filename)} %}"
      end
    end

    puts "=> Theme successfully switched!"
    puts "=> Reload your web-page to check it out =)"
  end # task :switch

  # Public: Install a theme using the theme packager.
  # Version 0.1.0 simple 1:1 file matching.
  #
  # git  - String, Optional path to the git repository of the theme to be installed.
  # name - String, Optional name of the theme you want to install.
  #        Passing name requires that the theme package already exist.
  #
  # Examples
  #
  #   rake theme:install git="https://github.com/jekyllbootstrap/theme-twitter.git"
  #   rake theme:install name="cool-theme"
  #
  # Returns Success/failure messages.
  desc "Install theme"
  task :install do
    if ENV["git"]
      manifest = theme_from_git_url(ENV["git"])
      name = manifest["name"]
    else
      name = ENV["name"].to_s.downcase
    end

    packaged_theme_path = JB::Path.build(:theme_packages, :node => name)

    abort("rake aborted!
      => ERROR: 'name' cannot be blank") if name.empty?
    abort("rake aborted!
      => ERROR: '#{packaged_theme_path}' directory not found.
      => Installable themes can be added via git. You can find some here: http://github.com/jekyllbootstrap
      => To download+install run: `rake theme:install git='[PUBLIC-CLONE-URL]'`
      => example : rake theme:install git='git@github.com:jekyllbootstrap/theme-the-program.git'
    ") unless FileTest.directory?(packaged_theme_path)

    manifest = verify_manifest(packaged_theme_path)

    # Get relative paths to packaged theme files
    # Exclude directories as they'll be recursively created. Exclude meta-data files.
    packaged_theme_files = []
    FileUtils.cd(packaged_theme_path) {
      Dir.glob("**/*.*") { |f|
        next if ( FileTest.directory?(f) || f =~ /^(manifest|readme|packager)/i )
        packaged_theme_files << f
      }
    }

    # Mirror each file into the framework making sure to prompt if already exists.
    packaged_theme_files.each do |filename|
      file_install_path = File.join(JB::Path.base, filename)
      if File.exist? file_install_path and ask("#{file_install_path} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
        next
      else
        mkdir_p File.dirname(file_install_path)
        cp_r File.join(packaged_theme_path, filename), file_install_path
      end
    end

    puts "=> #{name} theme has been installed!"
    puts "=> ---"
    if ask("=> Want to switch themes now?", ['y', 'n']) == 'y'
      system("rake switch_theme name='#{name}'")
    end
  end

  # Public: Package a theme using the theme packager.
  # The theme must be structured using valid JB API.
  # In other words packaging is essentially the reverse of installing.
  #
  # name - String, Required name of the theme you want to package.
  #
  # Examples
  #
  #   rake theme:package name="twitter"
  #
  # Returns Success/failure messages.
  desc "Package theme"
  task :package do
    name = ENV["name"].to_s.downcase
    theme_path = JB::Path.build(:themes, :node => name)
    asset_path = JB::Path.build(:theme_assets, :node => name)

    abort("rake aborted: name cannot be blank") if name.empty?
    abort("rake aborted: '#{theme_path}' directory not found.") unless FileTest.directory?(theme_path)
    abort("rake aborted: '#{asset_path}' directory not found.") unless FileTest.directory?(asset_path)

    ## Mirror theme's template directory (_includes)
    packaged_theme_path = JB::Path.build(:themes, :root => JB::Path.build(:theme_packages, :node => name))
    mkdir_p packaged_theme_path
    cp_r theme_path, packaged_theme_path

    ## Mirror theme's asset directory
    packaged_theme_assets_path = JB::Path.build(:theme_assets, :root => JB::Path.build(:theme_packages, :node => name))
    mkdir_p packaged_theme_assets_path
    cp_r asset_path, packaged_theme_assets_path

    ## Log packager version
    packager = {"packager" => {"version" => CONFIG["theme_package_version"].to_s } }
    open(JB::Path.build(:theme_packages, :node => "#{name}/packager.yml"), "w") do |page|
      page.puts packager.to_yaml
    end

    puts "=> '#{name}' theme is packaged and available at: #{JB::Path.build(:theme_packages, :node => name)}"
  end

end # end namespace :theme

namespace :literature do

# Public: Add a literature note template and create a timeline entry.
  #
  # doi - String, Required. URL DOI to article requested
  # date - String, optional. Reading date can provided manually (ISO-8601), otherwise current day is used.
  #
  # Examples
  #
  #   rake literature:add doi="10.1371/journal.pone.0000308"
  #
  # Returns Success/failure messages.
  desc "Add literature note."
  task :add do

    begin
      require 'cites'
    rescue LoadError
      abort("ERROR: gem cites must be installed")
    end

    require 'json'

    doi = ENV["doi"].to_s

    # Handle the requested DOI

    puts "Searching '#{doi}' from 'http://dx.doi.org'..."

    begin
      res = Cites.doi2cit(doi, format='citeproc-json', style='apa', locale='en-US', cache=FALSE)
    rescue Exception => e
      abort(e.message)
    end

    if res.length > 1
      puts "WARNING: found more than one match, which is strange..."
    end

    res = res[0]
    if !res.has_key?('title')
      title = ''
      STDOUT.puts "Found metadata which has no title, please enter the title manually"
      title = STDIN.gets.chomp
      res['title'] = title
    end

    if !res.has_key?('author')
      surname = ''
      STDOUT.puts "Found metadata which has no author, please enter the first author surname manually"
      surname = STDIN.gets.chomp
      res['author'] = [{}]
      res['author'][0]['family'] = surname
    end

    # Deal with the number of authors
    if res['author'].length == 1
      authors = "#{res['author'][0]['family']}"
      authors_underscore = authors
    elsif res['author'].length == 2
      authors = "#{res['author'][0]['family'] + ' and ' + res['author'][1]['family']}"
    else
      authors = "#{res['author'][0]['family']} et al."
    end

    year = res['issued']['date-parts'][0][0]

    puts authors

    puts "Found article '#{res['title']}' by #{authors} (#{year})"
    case ask("Is this correct?", ['y', 'n'])
      when 'Y', 'y','yes'
        puts "Proceeding"
      when /\A[nN]o?\Z/ #n or no
        abort("Quitting")
    end

    # Generate the note filename

    abort("rake aborted: '#{CONFIG['notes']}' directory not found.") unless FileTest.directory?(CONFIG['notes'])
    title = "#{authors.gsub("\.", "").gsub(" ", "_")}_#{year}"
    slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
    begin
      date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y-%m-%d')
    rescue => e
      puts "Error - date format must be YYYY-MM-DD, please check you typed it correctly!"
      exit -1
    end
    filename = File.join(CONFIG['notes'], "#{date}-#{slug}.#{CONFIG['post_ext']}")
    if File.exist?(filename)
      abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
    end

    # Generate the note template

    # Generate a image file name
    image_filename = File.join("img", authors.gsub("\.", "").gsub(" ", "_") + "_#{res['issued']['date-parts'][0][0]}" + ".jpg")

    puts "Creating new post: #{filename}"
    open(filename, 'w') do |post|
      post.puts "---"
      post.puts "layout: note"
      post.puts "name: \"#{date}-#{slug}\""
      post.puts "categories: \"reading\""
      post.puts "date: \"#{date}\""
      post.puts "title: \"#{res['title']}\""
      post.puts "authors: \"#{authors}\""
      post.puts "journal: \"#{res['container-title']}\""
      post.puts "year: \"#{year}\"" # "issued":{"date-parts":[[2007,3,21]]}
      post.puts "DOI: \"#{res['DOI']}\""
      post.puts "image: \"#{image_filename}\""
      post.puts "tags: []"
      post.puts "---"
      post.puts "{% include JB/setup %}"
      post.puts "\n### Abstract"
      post.puts "\n[PLACE ABSTRACT HERE]"
      post.puts "\n----"
      post.puts "\n### Notes"
      post.puts "\n[PLACE NOTES HERE]"
    end

    # Update literature.json
    lit_json = JSON.parse(File.read(CONFIG['lit_json']))

    date_json = (ENV['date'] ? Time.parse(ENV['date']) : Time.now).strftime('%Y,%-m,%d')

    text = [authors, "<br />", " Year: ", res['issued']['date-parts'][0][0], "<br />", " Journal: ",
            res["container-title"], "<br />", " DOI:", "<a href='http://dx.doi.org/#{res['DOI']}'>#{res['DOI']}</a>"].join(" ")

    require 'pp'
    pp res

    # Create a new hash to store the information on a given note
    note = { :startDate => date_json,
             :endDate   => date_json,
             :headline  => res['title'],
             :text      => text,
             :asset     => {
                              :media      => "notes/#{image_filename}",
                              :thumbnail  => "notes/#{image_filename}",
                              :credit     => "",
                              :caption    => ""
                           }
            }

    lit_json['timeline']['date'] << note
    no_entries = lit_json['timeline']['date'].length

    File.open(File.join(SOURCE, 'literature', 'literature.json'),"w") do |f|
      f.write(JSON.pretty_generate(lit_json))
    end

    # Set TimeLineJS to start from the latest entry
    target = "literature/literature_timeline.html"
    f = File.open(target, 'r+:UTF-8')
    new_file = f.read.sub(/(?<=start_at_slide:\s)[0-9]+/, "#{no_entries}")
    f.close
    FileUtils.rm(target)

    f = File.open(target, "w:UTF-8")
    f.write(new_file)
    f.close

  end # end task :add

  desc "Add literature note."
  task :test do


  end

end # end namespace :literature

namespace :site do

  desc "Generate blog files"
  task :generate do
    Jekyll::Site.new(Jekyll.configuration({
      "source"      => ".",
      "destination" => "_site",
      "url"         => "http://cbig.github.io/gpan-connectivity"
    })).process
  end

  desc "Run Jekyll server with production config"
  task :run => [:generate] do
    puts "Rakefile: Running Jekyll server with a production configuration."
    system "jekyll serve --watch --config _config.yml,_production_config.yml"
  end

  desc "Run Jekyll server with development config"
  task :run_dev => [:generate] do
    puts "Rakefile: Running Jekyll server with a development configuration."
    system "jekyll serve --watch --config _config.yml,_development_config.yml"
  end
end


# Internal: Download and process a theme from a git url.
# Notice we don't know the name of the theme until we look it up in the manifest.
# So we'll have to change the folder name once we get the name.
#
# url - String, Required url to git repository.
#
# Returns theme manifest hash
def theme_from_git_url(url)
  tmp_path = JB::Path.build(:theme_packages, :node => "_tmp")
  abort("rake aborted: system call to git clone failed") if !system("git clone #{url} #{tmp_path}")
  manifest = verify_manifest(tmp_path)
  new_path = JB::Path.build(:theme_packages, :node => manifest["name"])
  if File.exist?(new_path) && ask("=> #{new_path} theme package already exists. Override?", ['y', 'n']) == 'n'
    remove_dir(tmp_path)
    abort("rake aborted: '#{manifest["name"]}' already exists as theme package.")
  end

  remove_dir(new_path) if File.exist?(new_path)
  mv(tmp_path, new_path)
  manifest
end

# Internal: Process theme package manifest file.
#
# theme_path - String, Required. File path to theme package.
#
# Returns theme manifest hash
def verify_manifest(theme_path)
  manifest_path = File.join(theme_path, "manifest.yml")
  manifest_file = File.open( manifest_path )
  abort("rake aborted: repo must contain valid manifest.yml") unless File.exist? manifest_file
  manifest = YAML.load( manifest_file )
  manifest_file.close
  manifest
end

def ask(message, valid_options)
  if valid_options
    answer = get_stdin("#{message} #{valid_options.to_s.gsub(/"/, '').gsub(/, /,'/')} ") while !valid_options.include?(answer)
  else
    answer = get_stdin(message)
  end
  answer
end

def get_stdin(message)
  print message
  STDIN.gets.chomp
end

#Load custom rake scripts
Dir['_rake/*.rake'].each { |r| load r }
