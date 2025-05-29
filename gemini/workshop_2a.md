# TechDeck Workshop Part 2A: Loftwah Dataset & Images

Pull loftwah's complete data and process images with proper error handling.

## Prerequisites

Complete Part 1 and have the pipeline working with loftwah's basic data loaded.

## Exercise 1: Complete Loftwah Dataset Analysis

### Load and Explore Everything

```ruby
# Start Rails console
bin/rails console

# Load loftwah with all associations
user = User.includes(:tweets, :tags, :user_stats).find_by(handle: 'loftwah')

puts "🔍 Loftwah Complete Dataset"
puts "=" * 40

# Basic profile
puts "\n👤 PROFILE:"
puts "Name: #{user.name}"
puts "Handle: @#{user.handle}"
puts "Bio: #{user.bio}"
puts "Location: #{user.location}" if user.location
puts "Followers: #{user.followers.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse}"
puts "Following: #{user.following}"
puts "Follower ratio: #{(user.followers.to_f / [user.following, 1].max).round(2)}"
puts "Verified: #{user.verified? ? '✅' : '❌'}"
puts "Account created: #{user.account_created_at&.strftime('%B %Y')}" if user.account_created_at

# Image URLs
puts "\n🖼️ IMAGES:"
puts "Avatar (normal): #{user.avatar_twitter_normal_url}"
puts "Avatar (full): #{user.avatar_twitter_full_url}"
puts "Banner: #{user.banner_twitter_url}"
puts "Profile URL: #{user.profile_url}"
```

### Tweet Analysis by Type

```ruby
# Analyze tweet patterns
tweets = user.tweets.order(created_at_twitter: :desc)
puts "\n🐦 TWEETS ANALYSIS (#{tweets.count} total):"

# Group by type
tweet_types = tweets.group(:tweet_type).count
tweet_types.each do |type, count|
  percentage = (count.to_f / tweets.count * 100).round(1)
  puts "  #{type}: #{count} (#{percentage}%)"
end

# Separate collections for analysis
original_tweets = tweets.where(tweet_type: 'tweet')
replies = tweets.where(tweet_type: 'reply')
retweets = tweets.where(tweet_type: 'retweet')

puts "\n📊 CONTENT SAMPLES:"
puts "Original tweets (#{original_tweets.count}):"
original_tweets.limit(3).each_with_index do |tweet, i|
  puts "  #{i+1}. [#{tweet.likes} ❤️] #{tweet.text.truncate(80)}"
end

puts "\nReplies (#{replies.count}):"
replies.limit(3).each_with_index do |tweet, i|
  puts "  #{i+1}. [#{tweet.likes} ❤️] #{tweet.text.truncate(80)}"
end

# Engagement comparison
puts "\n📈 ENGAGEMENT BY TYPE:"
[
  ['Original Tweets', original_tweets],
  ['Replies', replies],
  ['Retweets', retweets]
].each do |name, collection|
  next if collection.empty?

  avg_likes = collection.average(:likes).to_f.round(2)
  avg_retweets = collection.average(:retweets).to_f.round(2)
  puts "  #{name}: #{avg_likes} avg likes, #{avg_retweets} avg RTs"
end

# Content characteristics
puts "\n📝 CONTENT CHARACTERISTICS:"
if original_tweets.any?
  puts "  Avg original tweet length: #{original_tweets.average('LENGTH(text)').to_f.round(0)} chars"
end
if replies.any?
  puts "  Avg reply length: #{replies.average('LENGTH(text)').to_f.round(0)} chars"
end

# Find most engaging content
best_tweet = tweets.order(likes: :desc).first
if best_tweet
  puts "  Best performing: #{best_tweet.likes} likes - \"#{best_tweet.text.truncate(60)}\""
end
```

## Exercise 2: Robust Image Processing

### URL Unshortening for t.co Links

```ruby
require 'net/http'
require 'uri'

# URL unshortener for t.co links
def unshorten_url(short_url, max_redirects: 5)
  return short_url unless short_url&.include?('t.co')

  puts "🔗 Unshortening: #{short_url}"

  begin
    uri = URI(short_url)
    redirects = 0

    loop do
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = (uri.scheme == 'https')
      http.open_timeout = 5
      http.read_timeout = 10

      # HEAD request to avoid downloading content
      request = Net::HTTP::Head.new(uri.request_uri)
      request['User-Agent'] = 'Mozilla/5.0 (compatible; TechDeck/1.0)'

      response = http.request(request)

      case response.code
      when '200'
        puts "   ✅ Final URL: #{uri}"
        return uri.to_s

      when '301', '302', '303', '307', '308'
        redirects += 1
        if redirects > max_redirects
          puts "   ❌ Too many redirects"
          return short_url
        end

        new_location = response['Location']
        if new_location
          uri = URI(new_location)
          puts "   🔄 → #{uri}"
        else
          return short_url
        end

      else
        puts "   ❌ HTTP #{response.code}"
        return short_url
      end
    end

  rescue => e
    puts "   ❌ Error: #{e.message}"
    return short_url
  end
end

# Process loftwah's URLs
puts "\n🔗 Processing URLs"
puts "=" * 20

# Check profile URL
if user.profile_url&.include?('t.co')
  puts "Profile URL: #{user.profile_url}"
  expanded_url = unshorten_url(user.profile_url)
  puts "Expanded: #{expanded_url}"

  if expanded_url != user.profile_url
    user.update(profile_url: expanded_url)
    puts "✅ Updated profile URL"
  end
end

# Check bio for t.co links
if user.bio&.include?('t.co')
  puts "\nProcessing bio links..."
  original_bio = user.bio

  updated_bio = user.bio.gsub(/https:\/\/t\.co\/\w+/) do |short_url|
    expanded = unshorten_url(short_url)
    puts "  #{short_url} → #{expanded}"
    expanded
  end

  if updated_bio != original_bio
    user.update(bio: updated_bio)
    puts "✅ Updated bio with expanded URLs"
  end
end
```

### Robust Image Downloader

```ruby
require 'marcel'
require 'base64'

# Robust image downloader that handles real-world issues
def download_image_robust(url, filename, max_redirects: 5)
  return nil unless url

  puts "📥 Downloading #{filename}..."
  puts "   URL: #{url}"

  begin
    uri = URI(url)
    redirects = 0

    loop do
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = (uri.scheme == 'https')
      http.open_timeout = 10
      http.read_timeout = 30

      # Set proper headers (Twitter images need User-Agent)
      request = Net::HTTP::Get.new(uri.request_uri)
      request['User-Agent'] = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
      request['Accept'] = 'image/webp,image/apng,image/*,*/*;q=0.8'

      response = http.request(request)

      case response.code
      when '200'
        if response.body && response.body.length > 0
          filepath = Rails.root.join('tmp', filename)
          File.write(filepath, response.body)

          mime_type = Marcel::MimeType.for(filepath)

          puts "   ✅ Downloaded: #{(response.body.length / 1024.0).round(2)}KB, #{mime_type}"

          return {
            data: response.body,
            filepath: filepath,
            mime_type: mime_type,
            size_kb: (response.body.length / 1024.0).round(2),
            final_url: uri.to_s
          }
        else
          puts "   ❌ Empty response body"
          return nil
        end

      when '301', '302', '303', '307', '308'
        redirects += 1
        if redirects > max_redirects
          puts "   ❌ Too many redirects (#{redirects})"
          return nil
        end

        new_location = response['Location']
        if new_location
          puts "   🔄 Redirecting to: #{new_location}"
          uri = URI(new_location)
        else
          puts "   ❌ Redirect without Location header"
          return nil
        end

      when '404'
        puts "   ❌ Not found (404) - Image may have been deleted"
        return nil

      when '403'
        puts "   ❌ Access forbidden (403)"
        return nil

      else
        puts "   ❌ HTTP #{response.code}: #{response.message}"
        return nil
      end
    end

  rescue => e
    puts "   ❌ Error: #{e.message}"
    return nil
  end
end

# Download loftwah's images
puts "\n🖼️ Downloading Loftwah's Images"
puts "=" * 35

avatar_data = download_image_robust(user.avatar_twitter_full_url, 'loftwah_avatar.jpg')
banner_data = download_image_robust(user.banner_twitter_url, 'loftwah_banner.jpg')

# Handle fallbacks
if !avatar_data && user.avatar_twitter_normal_url
  puts "\n🔄 Avatar failed, trying normal size..."
  avatar_data = download_image_robust(user.avatar_twitter_normal_url, 'loftwah_avatar_normal.jpg')
end

# Results summary
puts "\n📊 Download Results:"
if avatar_data
  puts "✅ Avatar: #{avatar_data[:size_kb]}KB, #{avatar_data[:mime_type]}"
  puts "   Gemini compatible: #{['image/jpeg', 'image/png', 'image/webp'].include?(avatar_data[:mime_type])}"
else
  puts "❌ Avatar download failed"
end

if banner_data
  puts "✅ Banner: #{banner_data[:size_kb]}KB, #{banner_data[:mime_type]}"
  puts "   Gemini compatible: #{['image/jpeg', 'image/png', 'image/webp'].include?(banner_data[:mime_type])}"
else
  puts "❌ Banner download failed"
end

# Store for next exercises
@loftwah_images = { avatar: avatar_data, banner: banner_data }
```

## Exercise 3: Mock Object Storage Upload

### Simulate Production Upload Workflow

```ruby
# Mock object storage service (simulates AWS S3, Google Cloud Storage, etc.)
class MockObjectStorage
  def self.upload(image_data, user_handle, image_type)
    return nil unless image_data

    # Generate storage path/filename
    extension = image_data[:mime_type].split('/').last
    filename = "#{user_handle}_#{image_type}.#{extension}"
    storage_path = "users/#{user_handle}/images/#{filename}"
    mock_url = "https://techdeck-storage.example.com/#{storage_path}"

    # Simulate upload process
    puts "📤 Mock Upload Process:"
    puts "   Local file: #{image_data[:filepath]}"
    puts "   Storage path: #{storage_path}"
    puts "   File size: #{image_data[:size_kb]}KB"
    puts "   MIME type: #{image_data[:mime_type]}"
    puts "   Final URL: #{mock_url}"

    # Simulate upload time based on file size
    upload_time = (image_data[:size_kb] / 100.0).round(2) # Mock: 100KB per second
    puts "   Upload time: #{upload_time}s (simulated)"

    # Return mock result
    {
      success: true,
      url: mock_url,
      storage_path: storage_path,
      upload_time: upload_time,
      file_size: image_data[:size_kb] * 1024
    }
  end

  def self.generate_signed_url(storage_path, expires_in: 3600)
    # Mock signed URL for temporary access
    base_url = "https://techdeck-storage.example.com"
    signature = "sig_#{Random.hex(16)}"
    expires_at = Time.current + expires_in

    "#{base_url}/#{storage_path}?signature=#{signature}&expires=#{expires_at.to_i}"
  end
end

# Upload loftwah's images to mock storage
puts "\n☁️ Mock Object Storage Upload"
puts "=" * 30

uploaded_images = {}

if @loftwah_images[:avatar]
  puts "\nUploading avatar..."
  result = MockObjectStorage.upload(@loftwah_images[:avatar], user.handle, 'avatar')
  uploaded_images[:avatar] = result
  puts "✅ Avatar uploaded to: #{result[:url]}"
end

if @loftwah_images[:banner]
  puts "\nUploading banner..."
  result = MockObjectStorage.upload(@loftwah_images[:banner], user.handle, 'banner')
  uploaded_images[:banner] = result
  puts "✅ Banner uploaded to: #{result[:url]}"
end

# Generate signed URLs for temporary access
puts "\n🔐 Generating Signed URLs (expires in 1 hour):"
uploaded_images.each do |type, data|
  if data
    signed_url = MockObjectStorage.generate_signed_url(data[:storage_path])
    puts "#{type.capitalize}: #{signed_url.truncate(80)}..."
  end
end

# Store URLs for database update (in production)
puts "\n💾 Database Update (Mock):"
avatar_storage_url = uploaded_images.dig(:avatar, :url)
banner_storage_url = uploaded_images.dig(:banner, :url)

puts "Would update user record with:"
puts "  avatar_storage_url: #{avatar_storage_url}" if avatar_storage_url
puts "  banner_storage_url: #{banner_storage_url}" if banner_storage_url

# In production, you'd do:
# user.update(
#   avatar_storage_url: avatar_storage_url,
#   banner_storage_url: banner_storage_url
# )
```

## Exercise 4: Gemini Image Analysis

### Send Images to Gemini (Following Docs Exactly)

```ruby
# Analyze images with Gemini using the exact pattern from your docs
def analyze_image_with_gemini(image_data, prompt_text)
  return nil unless image_data

  puts "🤖 Analyzing with Gemini..."

  # Encode image as base64
  base64_data = Base64.strict_encode64(image_data[:data])

  begin
    # Use stream_generate_content exactly as shown in your docs
    result = GEMINI_CLIENT.stream_generate_content({
      contents: [{
        role: 'user',
        parts: [
          { text: prompt_text },
          {
            inline_data: {
              mime_type: image_data[:mime_type],
              data: base64_data
            }
          }
        ]
      }]
    })

    # Process streaming response exactly as shown in your docs
    analysis_text = result
                    .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                    .map { |parts| parts.map { |part| part['text'] }.join }
                    .join

    puts "✅ Analysis completed"
    analysis_text

  rescue => e
    puts "❌ Gemini analysis failed: #{e.message}"
    nil
  end
end

# Analyze loftwah's avatar
if @loftwah_images[:avatar]
  puts "\n👤 Avatar Analysis"
  puts "=" * 20

  avatar_prompt = <<~PROMPT
    Analyze this Twitter avatar image for @#{user.handle} (#{user.name}).

    Context: This person has #{user.followers} followers and their bio says: "#{user.bio}"

    Describe:
    1. Visual style and presentation
    2. Professional/personal branding elements
    3. What this communicates about their tech expertise
    4. Personality indicators

    Keep response concise but insightful.
  PROMPT

  avatar_analysis = analyze_image_with_gemini(@loftwah_images[:avatar], avatar_prompt)
  puts avatar_analysis if avatar_analysis

  # Store for later use
  @avatar_analysis = avatar_analysis
end

# Analyze loftwah's banner
if @loftwah_images[:banner]
  puts "\n🎨 Banner Analysis"
  puts "=" * 20

  banner_prompt = <<~PROMPT
    Analyze this Twitter banner image for @#{user.handle}.

    Context: Tech professional with #{user.followers} followers

    Describe:
    1. Design elements and composition
    2. Brand messaging or themes
    3. How it complements their professional identity
    4. Technical or creative elements

    Keep response focused and practical.
  PROMPT

  banner_analysis = analyze_image_with_gemini(@loftwah_images[:banner], banner_prompt)
  puts banner_analysis if banner_analysis

  # Store for later use
  @banner_analysis = banner_analysis
end
```

### Combined Visual Analysis

```ruby
# Combine both image analyses for deeper insights
if @avatar_analysis || @banner_analysis
  puts "\n🔗 Combined Visual Brand Analysis"
  puts "=" * 35

  combined_prompt = <<~PROMPT
    Based on this user's visual branding, provide insights for their tech trading card:

    User: #{user.name} (@#{user.handle})
    Bio: #{user.bio}
    Followers: #{user.followers}

    Avatar Analysis: #{@avatar_analysis || 'No avatar available'}

    Banner Analysis: #{@banner_analysis || 'No banner available'}

    How do these visual elements support their technical persona?
    What does their visual branding suggest about their approach to technology?

    Keep response under 200 words.
  PROMPT

  # Standard text analysis (no JSON needed here)
  combined_result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: combined_prompt } }
  })

  # Process streaming response
  combined_analysis = combined_result
                      .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                      .map { |parts| parts.map { |part| part['text'] }.join }
                      .join

  puts combined_analysis
end
```

## Exercise 5: Data Export for Analysis

### Export Complete Dataset

```ruby
# Export loftwah's complete processed dataset
puts "\n📤 Exporting Complete Dataset"
puts "=" * 30

export_data = {
  profile: {
    name: user.name,
    handle: user.handle,
    bio: user.bio,
    location: user.location,
    followers: user.followers,
    following: user.following,
    verified: user.verified?,
    account_created: user.account_created_at,
    profile_url: user.profile_url,
    avatar_url: user.avatar_twitter_full_url,
    banner_url: user.banner_twitter_url
  },

  images: {
    avatar: @loftwah_images[:avatar] ? {
      size_kb: @loftwah_images[:avatar][:size_kb],
      mime_type: @loftwah_images[:avatar][:mime_type],
      local_path: @loftwah_images[:avatar][:filepath].to_s,
      storage_url: uploaded_images.dig(:avatar, :url)
    } : nil,

    banner: @loftwah_images[:banner] ? {
      size_kb: @loftwah_images[:banner][:size_kb],
      mime_type: @loftwah_images[:banner][:mime_type],
      local_path: @loftwah_images[:banner][:filepath].to_s,
      storage_url: uploaded_images.dig(:banner, :url)
    } : nil
  },

  content_analysis: {
    total_tweets: user.tweets.count,
    tweet_breakdown: user.tweets.group(:tweet_type).count,
    avg_likes: user.tweets.average(:likes).to_f.round(2),
    avg_retweets: user.tweets.average(:retweets).to_f.round(2),
    best_tweet: {
      text: user.tweets.order(likes: :desc).first&.text,
      likes: user.tweets.order(likes: :desc).first&.likes
    }
  },

  ai_analysis: {
    avatar_analysis: @avatar_analysis,
    banner_analysis: @banner_analysis
  },

  export_metadata: {
    exported_at: Time.current,
    rails_env: Rails.env,
    export_version: '2a'
  }
}

# Save to file
filename = "loftwah_dataset_part2a_#{Date.current.strftime('%Y%m%d')}.json"
filepath = Rails.root.join('tmp', filename)
File.write(filepath, JSON.pretty_generate(export_data))

puts "✅ Dataset exported to: #{filepath}"
puts "📊 Export includes:"
puts "   • Complete profile data with expanded URLs"
puts "   • Image processing results and storage URLs"
puts "   • Content analysis breakdown"
puts "   • AI-generated image descriptions"
puts "   • Export metadata for tracking"

# Quick summary
puts "\n📈 Part 2A Summary:"
puts "   Profile: #{user.name} (@#{user.handle})"
puts "   Images: #{@loftwah_images.count { |k,v| v }} downloaded successfully"
puts "   Content: #{user.tweets.count} tweets analyzed by type"
puts "   AI Analysis: #{[@avatar_analysis, @banner_analysis].compact.count} image descriptions"
puts "   Export: #{File.size(filepath)} bytes saved to #{filename}"
```

## Quick Reference

```ruby
# Part 2A Essential Commands
user = User.includes(:tweets, :tags).find_by(handle: 'loftwah')

# URL processing
expanded_url = unshorten_url(user.profile_url) if user.profile_url&.include?('t.co')

# Image download
avatar_data = download_image_robust(user.avatar_twitter_full_url, 'avatar.jpg')

# Mock storage
result = MockObjectStorage.upload(avatar_data, user.handle, 'avatar')

# Gemini analysis
analysis = analyze_image_with_gemini(avatar_data, "Analyze this avatar...")

# Export data
# File saved to tmp/loftwah_dataset_part2a_YYYYMMDD.json
```

**Part 2A Complete!**

You now have loftwah's complete dataset with:

- ✅ Profile data with expanded URLs
- ✅ Robust image downloading with fallbacks
- ✅ Mock object storage workflow
- ✅ Gemini image analysis using proper streaming
- ✅ Complete dataset export for further analysis
