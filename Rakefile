require 'pathname'
require 'base64'

ROOT   = Pathname(__FILE__).dirname
PUBLIC = ROOT.join('public')
SLIDES = ROOT.join('slides')
COMMON = ROOT.join('common')
VENDOR = ROOT.join('vendor')

require 'evil-front'
JqueryCdn.local_url = proc { '/jquery.js' }

class Pathname
  def glob(pattern, &block)
    Pathname.glob(self.join(pattern), &block)
  end

  def copy_to(to_dir, pattern = '**/*', &block)
    self.glob(pattern) do |from|
      next if from.directory?
      next if block_given? and yield
      to = to_dir.join(from.relative_path_from(self))
      to.dirname.mkpath
      FileUtils.cp(from, to)
    end
  end
end

Slide = Struct.new(:name, :title, :types, :html, :file) do
  def style
    file.dirname.basename.join("#{name}.css")
  end

  def name
    file.basename.sub_ext('')
  end
end

class Builder
  include EvilFront::Helpers

  attr_accessor :slides

  def initialize(build_type = :development)
    @build_type = build_type
  end

  def name(value);  @name = value; end
  def title(value); @title = value; end

  def type(*values)
    @types += ' ' + values.join(' ')
  end

  def render(file, &block)
    @current = file
    options  = { format: :html5, disable_escape: true }
    Slim::Template.new(file.to_s, options).render(self, &block)
  end

  def assets
    @sprockets ||= begin
      Sprockets::Environment.new(ROOT) do |env|
        env.append_path(SLIDES)
        env.append_path(COMMON)
        env.append_path(VENDOR)

        EvilFront.install_all(env)

        unless development?
          env.js_compressor  = Uglifier.new(copyright: false)
          env.css_compressor = :csso
        end
      end
    end
  end

  def slide(file)
    @name  = @title = @cover = nil
    @types = ''
    html = render(file)
    html = image_tag(@cover, class: 'cover') + html if @cover
    @slides << Slide.new(@name, @title, @types, html, file)
  end

  def slides_styles(&block)
    slides.map(&:style).reject {|i| assets[i].nil? }.each do |style|
      yield style
    end
  end

  def image_tag(name, attrs = { })
    attrs[:alt] ||= ''
    uri  = @current.dirname.join(name)
    type = file_type(uri)
    if type == 'image/gif'
      attrs[:class] = (attrs[:class] ? attrs[:class] + ' ' : '') + 'gif'
    end

    return uri.read if name =~ /\.svg$/

    if standalone?
      uri = encode_image(uri, type)
    else
      uri = uri.to_s.gsub(SLIDES.to_s, '')
      uri = './slides' + uri if production?
    end
    attrs = attrs.map { |k, v| "#{k}=\"#{v}\"" }.join(' ')
    "<img src=\"#{ uri }\" #{ attrs } />"
  end

  def encode_image(file, type)
    "data:#{type};base64," + Base64.encode64(file.open { |io| io.read })
  end

  def file_type(file)
    `file -ib #{file}`.split(';').first
  end

  def cover(name)
    @types += ' cover h'
    @cover  = name
  end

  def include_statistics
    COMMON.join('_statistics.html').read
  end

  def standalone?
    @build_type == :standalone
  end

  def production?
    @build_type == :production
  end

  def development?
    @build_type == :development
  end

  def result_file
    if standalone?
      'wsd2013.html'
    else
      'index.html'
    end
  end

  def layout_file
    COMMON.join('layout.slim')
  end

  def clean!
    PUBLIC.mkpath
    PUBLIC.glob('*') { |i| i.rmtree }
    self
  end

  def build!
    clean!

    @slides = []
    SLIDES.glob('**/*.slim').sort.each { |i| slide(i) }

    PUBLIC.join(result_file).open('w') { |io| io << render(layout_file) }
    if production?
      ROOT.copy_to(PUBLIC, '**/*.{png,gif,jpg}') do |image|
        image.to_s.start_with? PUBLIC.to_s
      end
    end

    if standalone?
      `zip -j public/wsd2013.zip public/wsd2013.html`
      FileUtils.rm PUBLIC.join('wsd2013.html')
    end

    self
  end
end

desc 'Build site files'
task :build do
  Builder.new(:production).build!
end

desc 'Build presentation all-in-one file'
task :standalone do
  Builder.new(:standalone).build!
end

desc 'Run server for development'
task :server do
  require 'sinatra/base'

  class WebSlides < Sinatra::Base
    set :lock, true

    get '/' do
      builder.build!
      send_file PUBLIC.join('index.html')
    end

    {
      css: 'text/css',  js:  'text/javascript',
      png: 'image/png', jpg: 'image/jpeg'
    }.each_pair do |ext, mime|
      get "/*.#{ ext }" do |path|
        path = path + ".#{ ext }"
        content_type mime
        builder.assets[path].to_s
      end
    end

    def builder
      @builder ||= Builder.new
    end
  end

  WebSlides.run!
end

desc 'Prepare commit to GitHub Pages'
task :deploy => :build do
  sh ['git checkout gh-pages',
      'git rm index.html',
      'git rm -r common/',
      'git rm -r slides/',
      'git rm -r vendor/',
      'cp -r public/* ./',
      'git add index.html',
      'git add common/',
      'git add slides/',
      'git add vendor/'].join(' && ')
end
