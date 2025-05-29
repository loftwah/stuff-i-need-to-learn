# SocialData.tools Twitter API Integration for Rails

Complete guide for fetching, storing, and analyzing Twitter data using the SocialData.tools API in Rails 8.

## API Overview

### Base URL & Authentication

```
Base URL: https://api.socialdata.tools
Authentication: Bearer token via Authorization header
Rate Limit: 120 requests/minute (shared across all endpoints)
```

### Available Endpoints

1. **User Profile**: `GET /twitter/user/[identifier]`
2. **User Tweets**: `GET /twitter/user/[user_id]/tweets`
3. **Tweets & Replies**: `GET /twitter/user/[user_id]/tweets-and-replies`

## Rails 8 Setup

### 1. Add HTTP Client Gem

```ruby
# Gemfile
gem 'faraday'
gem 'faraday-retry' # For automatic retries
```

### 2. Environment Configuration

```bash
# .env (development/test)
SOCIALDATA_API_KEY=your_api_key_here
```

```yaml
# config/credentials.yml.enc (production)
# EDITOR="code --wait" bin/rails credentials:edit
socialdata:
  api_key: your_production_api_key_here
```

### 3. Service Initializer

```ruby
# config/initializers/socialdata.rb
require 'faraday'
require 'faraday/retry'

SOCIALDATA_CLIENT = Faraday.new('https://api.socialdata.tools') do |conn|
  conn.request :json
  conn.request :retry, max: 3, interval: 0.5, backoff_factor: 2
  conn.response :json, content_type: /\bjson$/

  # Set authentication header
  api_key = Rails.env.production? ?
    Rails.application.credentials.dig(:socialdata, :api_key) :
    ENV['SOCIALDATA_API_KEY']

  conn.headers['Authorization'] = "Bearer #{api_key}"
  conn.headers['Accept'] = 'application/json'

  # Add request/response logging in development
  if Rails.env.development?
    conn.response :logger, Rails.logger, bodies: true
  end
end
```

## Core Service Classes

### 1. Twitter API Service

```ruby
# app/services/twitter_api_service.rb
class TwitterApiService
  class ApiError < StandardError; end
  class RateLimitError < ApiError; end
  class NotFoundError < ApiError; end

  def initialize
    @client = SOCIALDATA_CLIENT
  end

  # Fetch user profile by handle or ID
  def get_user_profile(identifier)
    response = @client.get("/twitter/user/#{identifier}")
    handle_response(response)
  rescue Faraday::Error => e
    handle_faraday_error(e, identifier)
  end

  # Fetch user tweets (2 pages with pagination)
  def get_user_tweets(user_id, pages: 2)
    all_tweets = []
    cursor = nil

    pages.times do |page|
      Rails.logger.info "Fetching tweets page #{page + 1} for user #{user_id}"

      url = "/twitter/user/#{user_id}/tweets-and-replies"
      url += "?cursor=#{CGI.escape(cursor)}" if cursor

      response = @client.get(url)
      data = handle_response(response)

      tweets = data['tweets'] || []
      all_tweets.concat(tweets)

      cursor = data['next_cursor']
      break unless cursor && tweets.any?

      # Rate limiting courtesy
      sleep(0.5) if page < pages - 1
    end

    all_tweets
  rescue Faraday::Error => e
    handle_faraday_error(e, user_id)
  end

  private

  def handle_response(response)
    case response.status
    when 200
      response.body
    when 402
      raise RateLimitError, "API quota exceeded"
    when 403
      raise ApiError, "User profile is protected"
    when 404
      raise NotFoundError, "User not found"
    when 422
      raise ApiError, "Invalid request parameters"
    when 500
      raise ApiError, "SocialData API server error - retry later"
    else
      raise ApiError, "Unexpected API response: #{response.status}"
    end
  end

  def handle_faraday_error(error, identifier)
    Rails.logger.error "Twitter API error for #{identifier}: #{error.message}"
    raise ApiError, "Failed to fetch data: #{error.message}"
  end
end
```

### 2. User Profile Service

```ruby
# app/services/user_profile_service.rb
class UserProfileService
  def initialize(twitter_api: TwitterApiService.new)
    @twitter_api = twitter_api
  end

  def create_or_update_user(identifier, submit_code: nil, submit_email: nil)
    # Fetch fresh data from API
    profile_data = @twitter_api.get_user_profile(identifier)

    # Process the profile data
    user_attributes = process_profile_data(profile_data)

    # Add submission tracking
    user_attributes[:submit_code] = submit_code if submit_code
    user_attributes[:submit_email] = submit_email if submit_email

    # Create or update user
    user = User.find_or_initialize_by(id_str: user_attributes[:id_str])

    # Track daily stats before updating
    track_daily_stats(user, user_attributes) if user.persisted?

    # Update user attributes
    user.assign_attributes(user_attributes)
    user.when_profile_last_updated = Time.current

    if user.save
      Rails.logger.info "Successfully updated user: #{user.handle}"
      user
    else
      Rails.logger.error "Failed to save user #{user.handle}: #{user.errors.full_messages}"
      raise "Failed to save user: #{user.errors.full_messages.join(', ')}"
    end
  end

  private

  def process_profile_data(data)
    # Process avatar URLs
    avatar_normal = data['profile_image_url_https']
    avatar_full = avatar_normal&.gsub('_normal', '') if avatar_normal

    # Process bio links (expand t.co URLs)
    processed_bio = process_bio_links(data['description'])

    {
      id_str: data['id_str'],
      name: data['name'],
      handle: data['screen_name'],
      location: data['location'],
      bio: processed_bio,
      followers: data['followers_count'],
      following: data['friends_count'],
      avatar_twitter_normal_url: avatar_normal,
      avatar_twitter_full_url: avatar_full,
      banner_twitter_url: data['profile_banner_url'],
      profile_url: data['url'],
      verified: data['verified'],
      account_created_at: parse_twitter_date(data['created_at'])
    }
  end

  def process_bio_links(bio)
    return bio unless bio

    # Find t.co links and expand them
    bio.gsub(/https:\/\/t\.co\/\w+/) do |short_url|
      expand_twitter_link(short_url) || short_url
    end
  end

  def expand_twitter_link(short_url)
    # This would need a separate service to expand URLs
    # For now, return the original URL
    # TODO: Implement URL expansion service
    short_url
  end

  def parse_twitter_date(date_string)
    return nil unless date_string
    Time.parse(date_string)
  rescue ArgumentError
    nil
  end

  def track_daily_stats(user, new_attributes)
    today = Date.current

    # Only create one record per day
    existing_stat = user.user_stats.find_by(recorded_on: today)
    return if existing_stat

    user.user_stats.create!(
      followers: new_attributes[:followers],
      following: new_attributes[:following],
      tweets_count: user.tweets.count, # Current count from DB
      recorded_on: today
    )
  end
end
```

### 3. Tweet Storage Service

```ruby
# app/services/tweet_storage_service.rb
class TweetStorageService
  def initialize(twitter_api: TwitterApiService.new)
    @twitter_api = twitter_api
  end

  def store_user_tweets(user_id_str, pages: 2)
    user = User.find_by!(id_str: user_id_str)

    # Fetch tweets from API
    tweets_data = @twitter_api.get_user_tweets(user_id_str, pages: pages)

    Rails.logger.info "Fetched #{tweets_data.length} tweets for user #{user.handle}"

    # Clear existing tweets for this user (fresh batch each time)
    user.tweets.destroy_all

    # Process and store tweets
    stored_count = 0
    tweets_data.each do |tweet_data|
      next unless tweet_data['id_str'] # Skip invalid tweets

      tweet_attributes = process_tweet_data(tweet_data, user_id_str)

      begin
        user.tweets.create!(tweet_attributes)
        stored_count += 1
      rescue ActiveRecord::RecordInvalid => e
        Rails.logger.warn "Failed to store tweet #{tweet_data['id_str']}: #{e.message}"
      end
    end

    Rails.logger.info "Stored #{stored_count} tweets for user #{user.handle}"
    stored_count
  end

  private

  def process_tweet_data(data, user_id_str)
    {
      user_id_str: user_id_str,
      tweet_id: data['id_str'],
      created_at_twitter: parse_twitter_date(data['tweet_created_at']),
      tweet_type: determine_tweet_type(data),
      text: data['full_text'],
      original_text: data['full_text'], # Store original for reference
      likes: data['favorite_count'] || 0,
      retweets: data['retweet_count'] || 0,
      quotes: data['quote_count'] || 0,
      replies: data['reply_count'] || 0,
      views: data['views_count'],
      bookmarks: data['bookmark_count'] || 0,
      media: process_media_urls(data),
      links: process_link_urls(data),
      lang: data['lang']
    }
  end

  def determine_tweet_type(data)
    return 'retweet' if data['full_text']&.start_with?('RT @')
    return 'reply' if data['in_reply_to_status_id_str']
    return 'quote' if data['quoted_status']
    'tweet'
  end

  def process_media_urls(data)
    media_entities = data.dig('entities', 'media') || []
    media_urls = media_entities.map { |m| m['media_url_https'] }.compact
    media_urls.any? ? media_urls : nil
  end

  def process_link_urls(data)
    url_entities = data.dig('entities', 'urls') || []
    expanded_urls = url_entities.map { |u| u['expanded_url'] }.compact
    expanded_urls.any? ? expanded_urls : nil
  end

  def parse_twitter_date(date_string)
    return nil unless date_string
    Time.parse(date_string)
  rescue ArgumentError
    nil
  end
end
```

## Rails Console Usage Examples

### Basic Profile Fetching

```ruby
# Start Rails console
bin/rails console

# Initialize services
twitter_api = TwitterApiService.new
profile_service = UserProfileService.new
tweet_service = TweetStorageService.new

# Fetch and create/update a user
user = profile_service.create_or_update_user('loftwah')
pp user.attributes

# Check the processed data
puts "User: #{user.name} (@#{user.handle})"
puts "Followers: #{user.followers}"
puts "Bio: #{user.bio}"
puts "Avatar (normal): #{user.avatar_twitter_normal_url}"
puts "Avatar (full): #{user.avatar_twitter_full_url}"
```

### Tweet Collection

```ruby
# Fetch and store tweets for a user
user = User.find_by(handle: 'loftwah')
tweet_count = tweet_service.store_user_tweets(user.id_str, pages: 2)

puts "Stored #{tweet_count} tweets"

# Examine the tweets
user.tweets.limit(5).each do |tweet|
  puts "#{tweet.created_at_twitter}: #{tweet.text}"
  puts "  Engagement: #{tweet.likes} likes, #{tweet.retweets} RTs"
  puts "  Type: #{tweet.tweet_type}"
  puts
end
```

### Data Analysis

```ruby
# Analyze tweet patterns
user = User.find_by(handle: 'loftwah')
tweets = user.tweets.order(created_at_twitter: :desc)

# Basic stats
puts "Total tweets: #{tweets.count}"
puts "Tweet types: #{tweets.group(:tweet_type).count}"
puts "Average likes: #{tweets.average(:likes).to_f.round(2)}"
puts "Most liked tweet: #{tweets.order(likes: :desc).first.text}"

# Engagement analysis
engagement_data = tweets.map do |tweet|
  total_engagement = tweet.likes + tweet.retweets + tweet.replies
  views = tweet.views || 1
  engagement_rate = (total_engagement.to_f / views * 100).round(2)

  {
    text: tweet.text.truncate(50),
    engagement_rate: engagement_rate,
    likes: tweet.likes,
    date: tweet.created_at_twitter
  }
end

# Top performing tweets
top_tweets = engagement_data.sort_by { |t| -t[:engagement_rate] }.first(5)
puts "\nTop performing tweets:"
top_tweets.each do |tweet|
  puts "#{tweet[:engagement_rate]}% - #{tweet[:text]}"
end
```

### Daily Stats Tracking

```ruby
# Check daily stats tracking
user = User.find_by(handle: 'loftwah')
recent_stats = user.user_stats.order(recorded_on: :desc).limit(7)

puts "Recent daily stats:"
recent_stats.each do |stat|
  puts "#{stat.recorded_on}: #{stat.followers} followers, #{stat.following} following"
end

# Calculate growth
if recent_stats.count >= 2
  latest = recent_stats.first
  previous = recent_stats.second
  follower_growth = latest.followers - previous.followers
  puts "Daily follower change: #{follower_growth > 0 ? '+' : ''}#{follower_growth}"
end
```

## Data Format Conversions

### Raw API Response Processing

```ruby
# Process raw API response in console
raw_profile = twitter_api.get_user_profile('loftwah')

# View raw JSON
puts JSON.pretty_generate(raw_profile)

# Extract specific fields using jq-style syntax
profile_summary = {
  name: raw_profile['name'],
  handle: raw_profile['screen_name'],
  location: raw_profile['location'],
  bio: raw_profile['description'],
  verified: raw_profile['verified'],
  followers: raw_profile['followers_count'],
  following: raw_profile['friends_count'],
  tweets: raw_profile['statuses_count'],
  likes: raw_profile['favourites_count'],
  listed: raw_profile['listed_count'],
  joined: raw_profile['created_at'],
  avatar: raw_profile['profile_image_url_https'],
  banner: raw_profile['profile_banner_url'],
  link: raw_profile['url']
}

puts JSON.pretty_generate(profile_summary)
```

### Tweet Data Processing

```ruby
# Process raw tweet data
raw_tweets = twitter_api.get_user_tweets('1192091185', pages: 1)

# Extract tweet essentials
tweet_summary = raw_tweets.map do |tweet|
  {
    id: tweet['id_str'],
    created_at: tweet['tweet_created_at'],
    type: tweet['full_text']&.start_with?('RT @') ? 'retweet' : 'tweet',
    text: tweet['full_text'],
    likes: tweet['favorite_count'],
    retweets: tweet['retweet_count'],
    quotes: tweet['quote_count'],
    replies: tweet['reply_count'],
    views: tweet['views_count'],
    bookmarks: tweet['bookmark_count'],
    lang: tweet['lang'],
    media: tweet.dig('entities', 'media')&.map { |m| m['media_url_https'] }&.compact,
    links: tweet.dig('entities', 'urls')&.map { |u| u['expanded_url'] }&.compact
  }
end

# Save to file for analysis
File.write('tweets_analysis.json', JSON.pretty_generate(tweet_summary))
puts "Saved #{tweet_summary.length} tweets to tweets_analysis.json"
```

## Advanced Workflows

### Batch User Processing

```ruby
# Process multiple users
handles = ['loftwah', 'metaprinxss', 'othertechuser']
results = {}

handles.each do |handle|
  begin
    Rails.logger.info "Processing #{handle}..."

    # Create/update user
    user = profile_service.create_or_update_user(handle)

    # Store tweets
    tweet_count = tweet_service.store_user_tweets(user.id_str)

    results[handle] = {
      success: true,
      user: user,
      tweet_count: tweet_count
    }

    # Rate limiting
    sleep(1)

  rescue => e
    Rails.logger.error "Failed to process #{handle}: #{e.message}"
    results[handle] = {
      success: false,
      error: e.message
    }
  end
end

# Summary
results.each do |handle, result|
  if result[:success]
    puts "✅ #{handle}: #{result[:tweet_count]} tweets stored"
  else
    puts "❌ #{handle}: #{result[:error]}"
  end
end
```

### Data Export for Analysis

```ruby
# Export user data for external analysis
users = User.includes(:tweets, :user_stats).limit(10)

export_data = users.map do |user|
  {
    profile: {
      name: user.name,
      handle: user.handle,
      followers: user.followers,
      following: user.following,
      bio: user.bio,
      verified: user.verified
    },
    tweets: user.tweets.limit(20).map do |tweet|
      {
        text: tweet.text,
        likes: tweet.likes,
        retweets: tweet.retweets,
        created_at: tweet.created_at_twitter,
        type: tweet.tweet_type
      }
    end,
    growth: user.user_stats.order(recorded_on: :desc).limit(30).map do |stat|
      {
        date: stat.recorded_on,
        followers: stat.followers,
        following: stat.following
      }
    end
  }
end

# Save as JSON
File.write('user_export.json', JSON.pretty_generate(export_data))
puts "Exported data for #{users.count} users"

# Convert to CSV for spreadsheet analysis
require 'csv'
CSV.open('users_summary.csv', 'w') do |csv|
  csv << ['Handle', 'Name', 'Followers', 'Following', 'Tweets', 'Avg Likes', 'Verified']

  users.each do |user|
    avg_likes = user.tweets.average(:likes)&.round(2) || 0
    csv << [
      user.handle,
      user.name,
      user.followers,
      user.following,
      user.tweets.count,
      avg_likes,
      user.verified
    ]
  end
end

puts "Exported CSV summary"
```

## Error Handling & Monitoring

### Robust Error Handling

```ruby
# Service with comprehensive error handling
class RobustTwitterService
  include Retryable

  def safe_user_update(handle, max_retries: 3)
    retryable(tries: max_retries, on: [TwitterApiService::RateLimitError]) do
      profile_service = UserProfileService.new
      profile_service.create_or_update_user(handle)
    end
  rescue TwitterApiService::NotFoundError
    Rails.logger.warn "User not found: #{handle}"
    nil
  rescue TwitterApiService::ApiError => e
    Rails.logger.error "API error for #{handle}: #{e.message}"
    raise
  rescue => e
    Rails.logger.error "Unexpected error for #{handle}: #{e.message}"
    raise
  end
end
```

### Rate Limiting Strategy

```ruby
# Rate-limited batch processing
class RateLimitedProcessor
  REQUESTS_PER_MINUTE = 100 # Stay under 120 limit

  def initialize
    @request_times = []
  end

  def process_users(handles)
    handles.each do |handle|
      wait_for_rate_limit

      begin
        user = UserProfileService.new.create_or_update_user(handle)
        TweetStorageService.new.store_user_tweets(user.id_str)
        Rails.logger.info "✅ Processed #{handle}"
      rescue => e
        Rails.logger.error "❌ Failed #{handle}: #{e.message}"
      end
    end
  end

  private

  def wait_for_rate_limit
    now = Time.current
    @request_times << now

    # Remove requests older than 1 minute
    @request_times.reject! { |time| time < 1.minute.ago }

    # If we're at the limit, wait
    if @request_times.count >= REQUESTS_PER_MINUTE
      sleep_time = 60 - (now - @request_times.first)
      sleep(sleep_time) if sleep_time > 0
    end
  end
end
```
