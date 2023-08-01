---
author: Aaron Breckenridge
title: Finding unused Rails view partials with DataDog and Github Actions
description: How BiggerPockets keeps track of unused view partials to make it easier to eliminiate dead code
tags: engineering rails
---

As a Rails application matures, views gets moved around and sometimes rewritten. This opens up the possibility of dead
view partials lying around: partials that are no longer rendered by any views. These unused view partials clutter up our
workspace and impose a small maintenance burden. At [BiggerPockets](https://www.biggerpockets.com), we use DataDog and
Github Actions to identify these unused partials so that they can be manually cleaned up.

# The script

This script lives in `bin/list-unused-partials`. When provided with the correct arguments, it connects to DataDog and
gets a list of all of the view partials that have been used in the last 90 days. It then compares that list of all of
the view partials in the Rails project. Finally, it returns a list of view partials that don't show up in DataDog:

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

require "net/http"
require "uri"
require "json"

# Get a list of unused view partials
class UnusedPartials
  attr_reader :options

  def initialize(**options)
    @options = options
  end

  # @return [Array<String>]
  def all
    get_all_partials - get_used_partials
  end

  private

  class HTTPError < StandardError; end

  def get_all_partials
    entries = Dir.glob(File.expand_path(File.join(File.dirname(__FILE__), "../app/views/**/_*.erb")))
    entries.map { |entry| entry.gsub(/.+\/app\/views/, "app/views") }
  end

  def get_used_partials
    uri = URI.parse("https://app.datadoghq.com/metric/flat_tags_for_metric?metric=trace.rails.render_partial.hits&filter%5B%5D=service%3A#{options[:datadog_service]}&filter%5B%5D=env%3Aproduction&window=#{2_592_000 * 3}&limit=10000")
    header = {
      "DD-API-KEY": options[:datadog_api_key],
      "DD-APPLICATION-KEY": options[:datadog_app_key],
      "Content-type": "application/json",
    }
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true
    request = Net::HTTP::Get.new(uri, header)
    response = http.request(request)
    raise HTTPError unless response.code == "200"
    JSON.parse(response.body)["tags"].select { |tag| tag =~ /^resource_name:/ }.map { |tag| tag.gsub(/^resource_name:/, "app/views/") }
  rescue HTTPError => error
    $stderr.puts("Error fetching list of view partials from DataDog: #{response.body}")
    raise error
  end
end

def main
  require "optparse"

  options = {}

  parser = OptionParser.new do |parser|
    parser.banner = <<~EOS
      List view partials not used in the last 90 days

      Usage: list-unused-partials [options]
    EOS

    parser.on("-a", "--datadog-api-key [STRING]", String, "Set the DataDog API key") do |val|
      options[:datadog_api_key] = val
    end
    parser.on("-p", "--datadog-app-key [STRING]", String, "Set the DataDog Application key") do |val|
      options[:datadog_app_key] = val
    end
    parser.on("-s", "--datadog-service [STRING]", String, "Set the DataDog Service name") do |val|
      options[:datadog_service] = val
    end
  end

  if ARGV.empty?
    puts parser
    exit 1
  end

  parser.parse!

  partials = UnusedPartials.new(**options).all
  puts partials
  exit partials.count > 0 ? 1 : 0
end

main if __FILE__ == $PROGRAM_NAME
```

Next up is a custom Github Action that runs once a week (or manually) that executes the above script. If there are any
unused view partials, the script exits with a non-zero code and the action fails.

Note that this script requires two secrets to be stored in Github:

- DATADOG_API_KEY
- DATADOG_APP_KEY

```yaml
name: Unused view partials

on:
  workflow_dispatch:
  schedule:
    - cron: "0 0 1 * *"

jobs:
  list_unused_partials:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 1

      - name: Detect unused partials
        run: bin/list-unused-partials -a "${{secrets.DATADOG_API_KEY}}" -p "${{secrets.DATADOG_APP_KEY}}" -s "biggerpockets"
```
