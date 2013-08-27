trap("SIGINT") { exit! }

task :default do
  exec 'rake -T'
end

def puts_and_exec command
  puts command
  exec command
end

desc 'Build site for production'
task :build do
  puts_and_exec 'jekyll build --config _config.yml,_config.production.yml,_data.yml --trace'
end

desc 'Preview and watch changes on local machine (development)'
task :preview do
  puts_and_exec 'jekyll serve --config _config.yml,_data.yml --watch --trace'
end

namespace :update do
  desc 'Update tweets'
  task :tweets do
    require 'yaml'

    secret = File.file?('_secret.yml') ? YAML.load_file('_secret.yml') : {}
    secret = {} if !secret.is_a?(Hash)
    consumer_key = secret['consumer_key'] || ""
    consumer_secret = secret['consumer_secret'] || ""
    oauth_token = secret['oauth_token'] || ""
    oauth_token_secret = secret['oauth_token_secret'] || ""

    url = 'https://api.twitter.com/1.1/statuses/user_timeline.json'

    require 'securerandom'

    parameters = {
      count: 5,
      screen_name: "caiguanhao",
    }
    oauth = {
      oauth_consumer_key: consumer_key,
      oauth_nonce: SecureRandom.base64(32).tr('+/=', 'abc'),
      oauth_signature_method: "HMAC-SHA1",
      oauth_timestamp: Time.now.to_i,
      oauth_token: oauth_token,
      oauth_version: "1.0",
    }

    require 'cgi'
    require 'openssl'
    require 'base64'

    # Creating Signature
    # https://dev.twitter.com/docs/auth/creating-signature
    oauth_signature_base = "GET&#{CGI::escape(url)}&"
    parameters.merge(oauth).sort.each{ |key, value| oauth_signature_base += "#{key}%3D#{CGI::escape(value.to_s)}%26" }
    oauth_signature_base.sub!(/%26$/, '')
    signing_key = "#{consumer_secret}&#{oauth_token_secret}"
    oauth[:oauth_signature] = CGI::escape(Base64.strict_encode64(OpenSSL::HMAC.digest('sha1', signing_key, oauth_signature_base)))
    oauth_header = "OAuth"
    oauth.each{ |key, value| oauth_header += " #{key}=\"#{value}\"," }
    oauth_header.sub!(/,$/, '')

    require 'net/https'
    uri = URI("#{url}?#{URI.encode_www_form(parameters)}")
    request = Net::HTTP::Get.new(uri)
    request['Authorization'] = oauth_header
    begin
      response = Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) do |http|
        http.request(request)
      end
    rescue EOFError
      puts response.inspect
      exit 1
    rescue OpenSSL::SSL::SSLError
      puts "You may need to run the following commands to temporarily fix the cert problem:"
      puts "  $ curl -Os http://curl.haxx.se/ca/cacert.pem"
      puts "  $ export SSL_CERT_FILE=#{File.expand_path('../cacert.pem', __FILE__)}"
      exit 1
    end
    if response.code != "200"
      puts response.inspect
      exit 1
    end

    data = YAML.load_file('_data.yml')
    data = {} if !data.is_a?(Hash)
    data['tweets'] = []

    require 'json'
    tweets = JSON.parse response.body
    tweets.each do |tweet|
      new_tweet = {}
      new_tweet['text'] = tweet['text']
      new_tweet['time'] = DateTime.parse(tweet['created_at']).strftime('%s').to_i
      tweet['entities']['urls'].each do |entity|
        new_tweet['text'].sub!(entity['url'], "<a href=\"#{entity['expanded_url']}\">#{entity['display_url']}</a>")
      end
      data['tweets'] << new_tweet
    end

    File.write('_data.yml', Psych.dump(data, line_width: -1))
    puts "#{tweets.size} tweets have been successfully saved to _data.yml."
  end
end
