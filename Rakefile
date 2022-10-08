require "faraday"
require "yaml"
require "uri"
require "logger"
require "fileutils"

class ReadmeRender
  def self.write(config_path: "config.yml", output_file: "README.md")
    new(config_path).write(output_file)
  end

  attr_reader :config

  def initialize(config_path)
    @config_path = config_path
    @config = YAML.load(File.read(@config_path))
  end

  def render
    markdown = ["# #{config["title"]}"]
    config["sections"].each do |section|
      render_section_title(section, markdown)
      render_section_alternatives(section, markdown)
    end

    markdown.join("\n\n")
  end

  def write(file)
    content = render
    logger.info "Writing #{file}"

    dirname = File.dirname(file)
    FileUtils.mkdir_p(dirname) if dirname != "."

    File.write(file, content)
  end

  private

  def render_section_title(section, markdown)
    name = section["name"]
    description = section["description"]
    name = name.to_s.empty? ? description : "`#{name}` #{description}"
    logger.info "Section: #{name}"

    markdown << "## #{name}"
  end

  def render_section_alternatives(section, markdown)
    markdown << section["alternatives"].each_with_object([]) do |alternative, obj|
      next unless item = render_alternative_metadata(alternative, markdown)

      logger.info " #{item}"
      obj << item
    end.join("\n")
  end

  def render_alternative_metadata(alternative, markdown)
    repo = alternative["repo"] ? Repo.new(alternative["repo"]) : nil
    name = alternative["name"] || repo&.name
    return if repo.to_s.empty? && name.to_s.empty?

    description = alternative["description"]
    if repo && description.to_s.empty?
      response = repo.repo_info
      logger.debug "(limit rate: #{response.headers["x-ratelimit-remaining"]}/#{response.headers["x-ratelimit-limit"]}, reset at #{Time.at(response.headers["x-ratelimit-reset"].to_i)})"
      description = response.body["description"]
    end

    "- [#{name}](#{repo&.url}): #{description}"
  end

  def logger
    @logger ||= -> {
      logger = Logger.new(STDOUT)
      logger.level = ENV["ACTIONS_RUNNER_DEBUG"] == "true" ? Logger::DEBUG : Logger::INFO
      logger
    }.call
  end

  class Repo
    attr_reader :url

    GITHUB_ENDPOINT = "https://api.github.com"

    def initialize(url)
      @url = URI.parse(url)
    end

    def repo_info
      @repo_info ||= send("#{source}_repo".to_sym)
    end

    def github_repo
      github_client.get("repos/#{repo_with_owner}")
    end

    def github_client
      @github_client ||= Faraday.new(GITHUB_ENDPOINT) do |f|
        f.headers["Accept"] = "application/vnd.github+json"
        f.request :url_encoded
        f.request :authorization, 'Bearer', github_token

        # f.response :logger
        f.response :json
      end
    end

    def github_token
      ENV["TOKEN_GITHUB"] || ENV["GITHUB_TOKEN"] || handle_missing_key
    end

    def handle_missing_key
      raise "Not found github token"
    end

    def source
      if github_repo?
        :github
      else
        :unknown
      end
    end

    def owner
      @owner ||= path.split("/").first
    end

    def name
      @name ||= path.split("/").last
    end

    def repo_with_owner
      @repo_with_owner ||= @url.path[1..-1]
    end

    def github_repo?
      @url.host == "github.com"
    end
  end
end

task :generate do
  ReadmeRender.write(output_file: "public/README.md")
end
