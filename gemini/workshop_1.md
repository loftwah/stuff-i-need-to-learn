# TechDeck Workshop Part 1: Controller Integration & Data Pipeline

Complete walkthrough of TechDeck's submission flow from controller to database, including SocialData API integration.

## System Architecture Overview

The TechDeck system follows this complete flow:

1. **User submits handle** via `SubmissionsController`
2. **Background job** triggers `ProfilePipelineService`
3. **Data fetching** via SocialData API services
4. **AI generation** via Gemini for card content
5. **Stats calculation** for RPG-style attributes

```
[Form] → [Controller] → [Background Job] → [Pipeline Service] → [API Services] → [AI Generation] → [Database]
```

## Prerequisites

Ensure you have both services initialized from previous guides:

- SocialData.tools API client (`SOCIALDATA_CLIENT`)
- Gemini AI client (`GEMINI_CLIENT`)

```bash
# Required gems
gem 'marcel'  # For MIME type detection
```

## Controller Integration Deep Dive

### Understanding the Submission Flow

The `SubmissionsController` is the entry point for TechDeck card creation:

```ruby
class SubmissionsController < ApplicationController
  def create
    handle = params[:handle]&.strip&.downcase
    submit_email = params[:submit_email]&.strip
    provided_code = params[:provided_submit_code]&.strip

    # Validation chain
    return handle_blank_error if handle.blank?
    return handle_invalid_code if provided_code != ENV["SUBMIT_CODE"]
    return handle_blacklisted if Blacklist.exists?(handle: handle)

    # This triggers the entire pipeline asynchronously
    SubmissionJob.perform_later(handle, submit_email, provided_code)

    flash[:notice] = "Submission received! Processing in background..."
    redirect_to new_submission_path
  end

  # Background processing entry point
  def self.process_submission_in_background(handle:, submit_email:, provided_code:)
    Rails.logger.info "Processing submission for handle: #{handle}"

    pipeline_service = ProfilePipelineService.new(
      handle: handle,
      submit_email: submit_email,
      provided_code: provided_code
    )

    result = pipeline_service.run
    handle_pipeline_result(result, handle)
  end
end
```

## Exercise 1: Complete Pipeline Simulation

### Build Our Own Pipeline Service

Since we may not have the complete `ProfilePipelineService`, let's build our own to understand the flow:

```ruby
# Start Rails console
bin/rails console

# Create a workshop pipeline service
class WorkshopPipelineService
  def initialize(handle:, submit_email: nil, provided_code: nil)
    @handle = handle
    @submit_email = submit_email
    @provided_code = provided_code
    @twitter_api = TwitterApiService.new
    @profile_service = UserProfileService.new
    @tweet_service = TweetStorageService.new
  end

  def run
    puts "🏃‍♂️ Starting pipeline for @#{@handle}"

    begin
      # Step 1: Fetch and store user profile
      puts "\n📊 Step 1: Fetching user profile..."
      user = @profile_service.create_or_update_user(
        @handle,
        submit_code: @provided_code,
        submit_email: @submit_email
      )
      log_user_creation(user)

      # Step 2: Fetch and store tweets
      puts "\n🐦 Step 2: Fetching tweets (2 pages)..."
      tweet_count = @tweet_service.store_user_tweets(user.id_str, pages: 2)
      puts "✅ Stored #{tweet_count} tweets"
      analyze_tweet_breakdown(user)

      # Step 3: Generate card stats
      puts "\n⚔️ Step 3: Calculating card stats..."
      user.reload # Ensure we have fresh tweets
      stats = generate_and_save_stats(user)

      # Step 4: Generate AI content
      puts "\n🤖 Step 4: Generating AI content..."
      ai_success = generate_ai_content(user)

      # Step 5: Final summary
      display_final_summary(user)

      { success: true, user: user, message: "Pipeline completed successfully" }

    rescue => e
      handle_pipeline_error(e)
    end
  end

  private

  def log_user_creation(user)
    puts "✅ User profile updated:"
    puts "   Name: #{user.name}"
    puts "   Handle: @#{user.handle}"
    puts "   Followers: #{format_number(user.followers)}"
    puts "   Following: #{user.following}"
    puts "   Bio: #{user.bio&.truncate(100)}"
    puts "   Avatar: #{user.avatar_twitter_full_url ? '✅' : '❌'}"
    puts "   Banner: #{user.banner_twitter_url ? '✅' : '❌'}"
  end

  def analyze_tweet_breakdown(user)
    tweet_types = user.tweets.group(:tweet_type).count
    puts "   Tweet breakdown:"
    tweet_types.each { |type, count| puts "     #{type}: #{count}" }

    # Show engagement stats
    tweets = user.tweets
    if tweets.any?
      avg_likes = tweets.average(:likes).to_f.round(2)
      avg_retweets = tweets.average(:retweets).to_f.round(2)
      puts "   Average engagement: #{avg_likes} likes, #{avg_retweets} retweets"
    end
  end

  def generate_and_save_stats(user)
    stats_service = CardStatService.new(user, tweets: user.tweets)
    stats = stats_service.generate_stats

    # Update user with stats
    user.update!(
      attack: stats[:attack],
      defense: stats[:defense],
      speed: stats[:speed],
      playing_card: stats[:playing_card],
      spirit_animal: stats[:spirit_animal]
    )

    puts "✅ Card stats calculated:"
    puts "   Attack: #{stats[:attack]} (followers + ratio influence)"
    puts "   Defense: #{stats[:defense]} (content production)"
    puts "   Speed: #{stats[:speed]} (posting frequency)"
    puts "   Playing Card: #{stats[:playing_card]}"
    puts "   Spirit Animal: #{stats[:spirit_animal]}"

    stats
  end

  def generate_ai_content(user)
    ai_service = AiGenerationService.new(user, tweets: user.tweets.limit(20))
    success = ai_service.generate!

    if success
      user.reload
      puts "✅ AI content generated successfully"
      puts "   Short bio length: #{user.short_bio&.length || 0} chars"
      puts "   Long bio length: #{user.long_bio&.length || 0} chars"
      puts "   Tags assigned: #{user.tags.count}"
      puts "   Buff: #{user.buff}"
      puts "   Weakness: #{user.weakness}"
      puts "   Vibe: #{user.vibe}"
      puts "   Special Move: #{user.special_move}"
    else
      puts "❌ AI content generation failed"
    end

    success
  end

  def display_final_summary(user)
    puts "\n🎯 Final Card Summary:"
    puts "=" * 50
    puts "#{user.name} (@#{user.handle})"
    puts "Stats: ATK=#{user.attack}, DEF=#{user.defense}, SPD=#{user.speed}"
    puts "Card: #{user.playing_card}"
    puts "Spirit: #{user.spirit_animal}"
    puts "Bio: #{user.short_bio&.truncate(100)}" if user.short_bio
    puts "Tags: #{user.tags.pluck(:name).join(', ')}" if user.tags.any?
    puts "Submission: #{@submit_email} (#{@provided_code})"
  end

  def format_number(num)
    num.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse
  end

  def handle_pipeline_error(error)
    puts "❌ Pipeline failed: #{error.message}"
    puts "   Error class: #{error.class.name}"
    puts "   Backtrace:"
    puts error.backtrace.first(5).map { |line| "     #{line}" }.join("\n")
    { success: false, error: error.message, backtrace: error.backtrace.first(5) }
  end
end

# Run the complete pipeline for loftwah
puts "🚀 Starting TechDeck Pipeline Workshop"
puts "=" * 60

pipeline = WorkshopPipelineService.new(
  handle: 'loftwah',
  submit_email: 'workshop@example.com',
  provided_code: 'workshop123'
)

result = pipeline.run

if result[:success]
  puts "\n🎉 Pipeline completed successfully!"
  user = result[:user]

  # Additional analysis
  puts "\n📊 Data Analysis:"
  puts "User ID: #{user.id_str}"
  puts "Profile last updated: #{user.when_profile_last_updated}"
  puts "Daily stats records: #{user.user_stats.count}"
  puts "Total tweets stored: #{user.tweets.count}"

  # Check for locked assignments
  locked = CardStatService::LOCKED_ASSIGNMENTS[user.handle&.downcase]
  if locked
    puts "🔒 Special user detected - locked assignments active"
  end

  boosted = CardStatService::BOOSTED_ASSIGNMENTS[user.handle&.downcase]
  if boosted
    puts "⚡ Boosted user detected - stat bonuses applied"
  end
else
  puts "\n💥 Pipeline failed!"
  puts "Error: #{result[:error]}"
  if result[:backtrace]
    puts "Debug info:"
    result[:backtrace].each { |line| puts "  #{line}" }
  end
end
```

## Exercise 2: Background Job Simulation

### Understanding Asynchronous Processing

```ruby
# The actual controller uses background jobs for processing
# Let's simulate this to understand the async flow

class WorkshopSubmissionJob
  def self.perform_later(handle, submit_email, provided_code)
    puts "📋 Job queued for: #{handle}"
    puts "   Email: #{submit_email || 'none provided'}"
    puts "   Code: #{provided_code ? '[PROVIDED]' : '[MISSING]'}"
    puts "   Timestamp: #{Time.current}"

    # In real Rails with Solid Queue/Sidekiq, this would be queued
    # For workshop, we'll process immediately to see the flow
    puts "🔄 Processing immediately (normally this would be async)..."
    perform_now(handle, submit_email, provided_code)
  end

  def self.perform_now(handle, submit_email, provided_code)
    puts "\n⚙️ Background job executing..."
    start_time = Time.current

    # This mirrors the controller's background method
    result = SubmissionsController.process_submission_in_background(
      handle: handle,
      submit_email: submit_email,
      provided_code: provided_code
    )

    end_time = Time.current
    duration = (end_time - start_time).round(2)

    puts "✅ Job completed in #{duration} seconds"
    result

  rescue => e
    puts "❌ Job failed after #{((Time.current - start_time)).round(2)}s"
    handle_job_failure(handle, e)
  end

  def self.handle_job_failure(handle, error)
    puts "📧 Job failure handling for #{handle}:"
    puts "   Error: #{error.message}"
    puts "   Error class: #{error.class.name}"

    # In production, this would:
    # - Log to error tracking (Sentry, Bugsnag, etc.)
    # - Send email notification to user
    # - Potentially retry with exponential backoff
    # - Update submission status in database

    puts "   🔄 Would implement retry logic"
    puts "   📧 Would send user notification"
    puts "   📊 Would log to error tracking service"
  end
end

# Simulate the complete controller → job → pipeline flow
def simulate_form_submission(handle:, email: nil, code: nil)
  puts "\n📝 Form Submission Simulation"
  puts "Handle: #{handle}"
  puts "Email: #{email || 'none'}"
  puts "Code: #{code || 'none'}"
  puts "-" * 40

  # Step 1: Controller validation (simplified)
  puts "🔍 Running controller validations..."

  if handle.blank?
    puts "❌ Validation failed: Handle cannot be blank"
    return { success: false, error: "Handle cannot be blank" }
  end

  if code.blank?
    puts "❌ Validation failed: Submission code required"
    return { success: false, error: "Submission code required" }
  end

  # Check against environment variable (if set)
  expected_code = ENV["SUBMIT_CODE"]
  if expected_code.present? && code != expected_code
    puts "❌ Validation failed: Invalid submission code"
    return { success: false, error: "Invalid submission code" }
  end

  if Blacklist.exists?(handle: handle)
    puts "❌ Validation failed: Handle is blacklisted"
    return { success: false, error: "Handle is blacklisted" }
  end

  puts "✅ All validations passed"

  # Step 2: Queue background job
  puts "\n🚀 Queueing background job..."
  job_result = WorkshopSubmissionJob.perform_later(handle, email, code)

  { success: true, message: "Submission processed", job_result: job_result }
end

# Test different submission scenarios
test_scenarios = [
  {
    name: "Valid Submission",
    handle: 'loftwah',
    email: 'loftwah@example.com',
    code: 'valid123'
  },
  {
    name: "Empty Handle",
    handle: '',
    email: 'empty@example.com',
    code: 'test123'
  },
  {
    name: "Missing Code",
    handle: 'testuser',
    email: 'test@example.com',
    code: nil
  },
  {
    name: "Non-existent User",
    handle: 'nonexistentuser12345xyz',
    email: 'fake@example.com',
    code: 'test123'
  }
]

test_scenarios.each_with_index do |scenario, index|
  puts "\n" + "=" * 70
  puts "TEST #{index + 1}: #{scenario[:name]}"
  puts "=" * 70

  result = simulate_form_submission(
    handle: scenario[:handle],
    email: scenario[:email],
    code: scenario[:code]
  )

  puts "\n📊 Final Result:"
  if result[:success]
    puts "✅ SUCCESS: #{result[:message]}"
  else
    puts "❌ FAILED: #{result[:error]}"
  end

  # Rate limiting between tests
  sleep(2) if index < test_scenarios.length - 1
end
```

## Exercise 3: Data Analysis and Validation

### Analyzing the Complete User Dataset

```ruby
# After running the pipeline, let's analyze what we've collected
user = User.find_by(handle: 'loftwah')

puts "🔍 Complete Data Analysis for @#{user.handle}"
puts "=" * 60

# Profile Data Analysis
puts "\n👤 PROFILE DATA:"
puts "ID: #{user.id_str}"
puts "Name: #{user.name}"
puts "Location: #{user.location}"
puts "Bio length: #{user.bio&.length || 0} characters"
puts "Followers: #{format_number(user.followers)}"
puts "Following: #{user.following}"
puts "Follower ratio: #{(user.followers.to_f / [user.following, 1].max).round(2)}"
puts "Account age: #{((Time.current - user.account_created_at) / 1.year).round(1)} years" if user.account_created_at
puts "Verified: #{user.verified? ? '✅' : '❌'}"

# Image Data Analysis
puts "\n🖼️ IMAGE DATA:"
puts "Avatar (normal): #{user.avatar_twitter_normal_url ? '✅' : '❌'}"
puts "Avatar (full): #{user.avatar_twitter_full_url ? '✅' : '❌'}"
puts "Banner: #{user.banner_twitter_url ? '✅' : '❌'}"
puts "Profile URL: #{user.profile_url || 'none'}"

# Tweet Data Analysis
tweets = user.tweets.order(created_at_twitter: :desc)
puts "\n🐦 TWEET DATA:"
puts "Total tweets stored: #{tweets.count}"

if tweets.any?
  tweet_types = tweets.group(:tweet_type).count
  puts "Tweet breakdown:"
  tweet_types.each { |type, count| puts "  #{type}: #{count}" }

  # Engagement analysis
  total_likes = tweets.sum(:likes)
  total_retweets = tweets.sum(:retweets)
  total_replies = tweets.sum(:replies)

  puts "Engagement totals:"
  puts "  Likes: #{format_number(total_likes)}"
  puts "  Retweets: #{format_number(total_retweets)}"
  puts "  Replies: #{format_number(total_replies)}"

  puts "Engagement averages:"
  puts "  Avg likes: #{tweets.average(:likes).to_f.round(2)}"
  puts "  Avg retweets: #{tweets.average(:retweets).to_f.round(2)}"
  puts "  Avg replies: #{tweets.average(:replies).to_f.round(2)}"

  # Time analysis
  if tweets.count >= 2
    first_tweet = tweets.last
    latest_tweet = tweets.first
    time_span = (latest_tweet.created_at_twitter - first_tweet.created_at_twitter) / 1.day
    puts "Tweet timespan: #{time_span.round(1)} days"
    puts "Tweet frequency: #{(tweets.count / (time_span + 1)).round(3)} tweets/day"
  end
end

# Card Stats Analysis
puts "\n⚔️ CARD STATS:"
puts "Attack: #{user.attack} (influence & reach)"
puts "Defense: #{user.defense} (content consistency)"
puts "Speed: #{user.speed} (activity frequency)"
puts "Playing Card: #{user.playing_card}"
puts "Spirit Animal: #{user.spirit_animal}"

# Check for special assignments
locked = CardStatService::LOCKED_ASSIGNMENTS[user.handle&.downcase]
boosted = CardStatService::BOOSTED_ASSIGNMENTS[user.handle&.downcase]

if locked
  puts "🔒 LOCKED ASSIGNMENTS:"
  puts "  Spirit Animal: #{locked[:spirit_animal]}"
  puts "  Playing Card: #{locked[:playing_card]}"
end

if boosted
  puts "⚡ STAT BOOSTS:"
  boosted.each { |stat, boost| puts "  #{stat.capitalize}: +#{boost}" }
end

# AI-Generated Content Analysis
puts "\n🤖 AI-GENERATED CONTENT:"
puts "Short bio: #{user.short_bio&.length || 0}/250 chars"
puts "Long bio: #{user.long_bio&.length || 0}/1100 chars"
puts "Tags: #{user.tags.count}/6 assigned"

if user.tags.any?
  puts "  Tags: #{user.tags.pluck(:name).join(', ')}"
end

puts "Buff: #{user.buff} (#{user.buff&.length || 0}/30 chars)"
puts "Weakness: #{user.weakness} (#{user.weakness&.length || 0}/30 chars)"
puts "Vibe: #{user.vibe} (#{user.vibe&.length || 0}/30 chars)"
puts "Special Move: #{user.special_move} (#{user.special_move&.length || 0}/30 chars)"
puts "Flavor Text: #{user.flavor_text&.length || 0}/60 chars"

# Descriptions
puts "\nDescriptions:"
puts "  Buff: #{user.buff_description&.length || 0}/100 chars"
puts "  Weakness: #{user.weakness_description&.length || 0}/100 chars"
puts "  Vibe: #{user.vibe_description&.length || 0}/100 chars"
puts "  Special Move: #{user.special_move_description&.length || 0}/100 chars"

# Submission tracking
puts "\n📋 SUBMISSION DATA:"
puts "Submit email: #{user.submit_email || 'none'}"
puts "Submit code: #{user.submit_code || 'none'}"
puts "Profile claimed: #{user.profile_claimed? ? '✅' : '❌'}"
puts "Last updated: #{user.when_profile_last_updated}"

# Daily stats tracking
recent_stats = user.user_stats.order(recorded_on: :desc).limit(7)
puts "\n📈 DAILY STATS (last 7 days):"
if recent_stats.any?
  recent_stats.each do |stat|
    puts "  #{stat.recorded_on}: #{format_number(stat.followers)} followers, #{stat.following} following"
  end

  if recent_stats.count >= 2
    growth = recent_stats.first.followers - recent_stats.last.followers
    days = (recent_stats.first.recorded_on - recent_stats.last.recorded_on).to_i
    puts "  Growth: #{growth > 0 ? '+' : ''}#{growth} followers over #{days} days"
  end
else
  puts "  No daily stats recorded yet"
end

def format_number(num)
  num.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse
end
```

### Data Quality Validation

```ruby
# Validate the quality of our collected data
def validate_user_data(user)
  puts "\n🔬 Data Quality Validation for @#{user.handle}"
  puts "=" * 50

  issues = []
  warnings = []

  # Profile completeness
  issues << "Missing name" if user.name.blank?
  issues << "Missing bio" if user.bio.blank?
  warnings << "No location specified" if user.location.blank?
  warnings << "No profile URL" if user.profile_url.blank?

  # Image availability
  issues << "Missing avatar URL" if user.avatar_twitter_normal_url.blank?
  warnings << "Missing banner URL" if user.banner_twitter_url.blank?
  warnings << "No full-size avatar URL" if user.avatar_twitter_full_url.blank?

  # Stats validation
  issues << "Missing attack stat" if user.attack.nil?
  issues << "Missing defense stat" if user.defense.nil?
  issues << "Missing speed stat" if user.speed.nil?
  issues << "Missing playing card" if user.playing_card.blank?
  issues << "Missing spirit animal" if user.spirit_animal.blank?

  # AI content validation
  if user.short_bio.present?
    if user.short_bio.length < 150
      issues << "Short bio too short (#{user.short_bio.length}/150 min)"
    elsif user.short_bio.length > 250
      issues << "Short bio too long (#{user.short_bio.length}/250 max)"
    end
  else
    issues << "Missing short bio"
  end

  if user.long_bio.present?
    if user.long_bio.length < 900
      warnings << "Long bio might be too short (#{user.long_bio.length}/900 min)"
    elsif user.long_bio.length > 1100
      issues << "Long bio too long (#{user.long_bio.length}/1100 max)"
    end
  else
    issues << "Missing long bio"
  end

  # Tags validation
  tag_count = user.tags.count
  if tag_count == 0
    issues << "No tags assigned"
  elsif tag_count < 6
    warnings << "Only #{tag_count}/6 tags assigned"
  elsif tag_count > 6
    issues << "Too many tags assigned (#{tag_count}/6 max)"
  end

  # Character limit validations
  [
    [:buff, 30],
    [:weakness, 30],
    [:vibe, 30],
    [:special_move, 30],
    [:buff_description, 100],
    [:weakness_description, 100],
    [:vibe_description, 100],
    [:special_move_description, 100]
  ].each do |field, max_length|
    value = user.send(field)
    if value.present? && value.length > max_length
      issues << "#{field.to_s.humanize} too long (#{value.length}/#{max_length} max)"
    elsif value.blank?
      issues << "Missing #{field.to_s.humanize.downcase}"
    end
  end

  # Flavor text special validation (30-60 chars)
  if user.flavor_text.present?
    length = user.flavor_text.length
    if length < 30
      issues << "Flavor text too short (#{length}/30 min)"
    elsif length > 60
      issues << "Flavor text too long (#{length}/60 max)"
    end
  else
    issues << "Missing flavor text"
  end

  # Tweet data validation
  tweet_count = user.tweets.count
  if tweet_count == 0
    issues << "No tweets stored"
  elsif tweet_count < 10
    warnings << "Only #{tweet_count} tweets stored (may affect AI quality)"
  end

  # Display results
  puts "\n✅ VALIDATION RESULTS:"
  if issues.empty? && warnings.empty?
    puts "🎉 Perfect! No issues found."
  else
    if issues.any?
      puts "\n❌ CRITICAL ISSUES (#{issues.count}):"
      issues.each { |issue| puts "  • #{issue}" }
    end

    if warnings.any?
      puts "\n⚠️ WARNINGS (#{warnings.count}):"
      warnings.each { |warning| puts "  • #{warning}" }
    end
  end

  # Overall score
  total_checks = 25 # Approximate number of validations
  critical_weight = 2
  warning_weight = 1

  penalty = (issues.count * critical_weight) + (warnings.count * warning_weight)
  score = [((total_checks - penalty).to_f / total_checks * 100).round, 0].max

  puts "\n📊 OVERALL DATA QUALITY SCORE: #{score}%"

  case score
  when 90..100
    puts "🌟 Excellent - Ready for production"
  when 80..89
    puts "👍 Good - Minor improvements needed"
  when 70..79
    puts "⚠️ Fair - Several issues to address"
  when 60..69
    puts "❌ Poor - Major issues need fixing"
  else
    puts "💥 Critical - Data unusable"
  end

  {
    score: score,
    issues: issues,
    warnings: warnings,
    ready_for_production: issues.empty?
  }
end

# Run validation on our user
validation_result = validate_user_data(user)

# If there are issues, let's try to fix some automatically
if validation_result[:issues].any?
  puts "\n🔧 Attempting automatic fixes..."

  # Re-run AI generation if content is missing/invalid
  missing_ai_content = validation_result[:issues].any? { |issue|
    issue.include?('bio') || issue.include?('tags') || issue.include?('buff')
  }

  if missing_ai_content
    puts "🤖 Re-running AI generation to fix content issues..."
    ai_service = AiGenerationService.new(user, tweets: user.tweets.limit(20))
    success = ai_service.generate!

    if success
      puts "✅ AI generation completed - re-validating..."
      user.reload
      new_validation = validate_user_data(user)

      improvement = new_validation[:score] - validation_result[:score]
      puts "📈 Score improvement: +#{improvement}% (#{validation_result[:score]}% → #{new_validation[:score]}%)"
    else
      puts "❌ AI generation failed"
    end
  end
end
```

## Exercise 4: Image Processing and Analysis

### Download and Analyze Profile Images

```ruby
require 'marcel'
require 'net/http'
require 'uri'
require 'base64'

# Helper to download and analyze images
def download_and_analyze_image(url, filename)
  return nil unless url

  puts "📥 Downloading #{filename}..."

  begin
    # Download image
    uri = URI(url)
    response = Net::HTTP.get_response(uri)

    if response.code == '200'
      # Save locally in tmp directory
      filepath = Rails.root.join('tmp', filename)
      File.write(filepath, response.body)

      # Detect MIME type using Marcel
      mime_type = Marcel::MimeType.for(filepath)
      file_size = File.size(filepath)

      puts "✅ Downloaded #{filename}"
      puts "   MIME type: #{mime_type}"
      puts "   File size: #{(file_size / 1024.0).round(2)} KB"
      puts "   Local path: #{filepath}"

      {
        filepath: filepath,
        mime_type: mime_type,
        file_size: file_size,
        data: response.body,
        url: url
      }
    else
      puts "❌ Failed to download #{filename}: HTTP #{response.code}"
      nil
    end
  rescue => e
    puts "❌ Error downloading #{filename}: #{e.message}"
    nil
  end
end

# Download loftwah's images
user = User.find_by(handle: 'loftwah')
puts "🖼️ Processing images for @#{user.handle}"

# Download avatar (full size)
avatar_data = download_and_analyze_image(
  user.avatar_twitter_full_url,
  "#{user.handle}_avatar.jpg"
)

# Download banner
banner_data = download_and_analyze_image(
  user.banner_twitter_url,
  "#{user.handle}_banner.jpg"
)

# Store for later use
@user_images = {
  avatar: avatar_data,
  banner: banner_data
}

# Validate image data
puts "\n🔍 Image Validation:"
if avatar_data
  puts "✅ Avatar processed successfully"
  puts "   Supported for Gemini: #{['image/jpeg', 'image/png', 'image/webp'].include?(avatar_data[:mime_type])}"
else
  puts "❌ Avatar processing failed"
end

if banner_data
  puts "✅ Banner processed successfully"
  puts "   Supported for Gemini: #{['image/jpeg', 'image/png', 'image/webp'].include?(banner_data[:mime_type])}"
else
  puts "❌ Banner processing failed or no banner URL"
end

# Mock object storage upload
def mock_upload_to_storage(image_data, user_handle, image_type)
  return nil unless image_data

  # In production, this would upload to AWS S3, Google Cloud Storage, etc.
  filename = "#{user_handle}_#{image_type}.#{image_data[:mime_type].split('/').last}"
  mock_url = "https://techdeck-storage.example.com/images/#{filename}"

  puts "📤 Mock upload: #{filename} → #{mock_url}"
  puts "   Size: #{(image_data[:file_size] / 1024.0).round(2)} KB"
  puts "   MIME: #{image_data[:mime_type]}"

  # Return mock storage URL
  mock_url
end

if @user_images[:avatar]
  avatar_storage_url = mock_upload_to_storage(@user_images[:avatar], user.handle, 'avatar')
  puts "Avatar would be stored at: #{avatar_storage_url}"
end

if @user_images[:banner]
  banner_storage_url = mock_upload_to_storage(@user_images[:banner], user.handle, 'banner')
  puts "Banner would be stored at: #{banner_storage_url}"
end
```

### Send Images to Gemini for Analysis

```ruby
# Analyze images with Gemini
def analyze_image_with_gemini(image_data, image_type, user_context = nil)
  return nil unless image_data

  puts "🤖 Analyzing #{image_type} with Gemini..."

  # Encode image as base64
  base64_data = Base64.strict_encode64(image_data[:data])

  # Create context-aware prompt
  context = user_context ? "\n\nUser context: #{user_context}" : ""

  prompt = case image_type
  when :avatar
    <<~PROMPT
      Analyze this Twitter profile avatar image. Describe:
      1. The person's appearance and style
      2. Professional presentation and branding
      3. What this communicates about their personal brand
      4. Technical or creative elements visible
      5. Overall impression and personality indicators#{context}
    PROMPT
  when :banner
    <<~PROMPT
      Analyze this Twitter banner image. Describe:
      1. Visual design elements and composition
      2. Colors, typography, and branding
      3. Technical or professional themes
      4. What this communicates about the person
      5. How it complements their professional identity#{context}
    PROMPT
  end

  begin
    result = GEMINI_CLIENT.stream_generate_content({
      contents: [{
        role: 'user',
        parts: [
          { text: prompt },
          {
            inline_data: {
              mime_type: image_data[:mime_type],
              data: base64_data
            }
          }
        ]
      }]
    })

    # Extract text from streaming response
    analysis_text = result.map { |response|
      response.dig('candidates', 0, 'content', 'parts', 0, 'text')
    }.compact.join

    puts "✅ #{image_type.capitalize} analysis completed"
    analysis_text
  rescue => e
    puts "❌ Failed to analyze #{image_type}: #{e.message}"
    nil
  end
end

# Analyze both images with user context
user_context = "User: #{user.name} (@#{user.handle}), #{user.bio&.truncate(100)}"

if @user_images[:avatar]
  puts "\n👤 Avatar Analysis:"
  puts "=" * 40
  avatar_analysis = analyze_image_with_gemini(@user_images[:avatar], :avatar, user_context)
  puts avatar_analysis if avatar_analysis

  # Store analysis for later use
  @avatar_analysis = avatar_analysis
end

if @user_images[:banner]
  puts "\n🎨 Banner Analysis:"
  puts "=" * 40
  banner_analysis = analyze_image_with_gemini(@user_images[:banner], :banner, user_context)
  puts banner_analysis if banner_analysis

  # Store analysis for later use
  @banner_analysis = banner_analysis
end

# Combine image analysis with other data
if @avatar_analysis || @banner_analysis
  puts "\n🔗 Combined Visual Brand Analysis:"

  combined_prompt = <<~PROMPT
    Based on this user's visual branding, provide insights for their tech trading card:

    User: #{user.name} (@#{user.handle})
    Bio: #{user.bio}
    Stats: ATK=#{user.attack}, DEF=#{user.defense}, SPD=#{user.speed}

    Avatar Analysis: #{@avatar_analysis || 'No avatar analysis available'}

    Banner Analysis: #{@banner_analysis || 'No banner analysis available'}

    How do these visual elements support or enhance their technical persona?
    What does their visual branding suggest about their approach to technology and community?
  PROMPT

  # Get JSON response for structured insights
  result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: combined_prompt } },
    generation_config: { response_mime_type: 'application/json' }
  })

  json_string = result
                .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                .map { |parts| parts.map { |part| part['text'] }.join }
                .join

  visual_insights = JSON.parse(json_string)
  puts JSON.pretty_generate(visual_insights)
end
```

## Cheat Sheet: Controller → Pipeline Flow

### Quick Commands Reference

```ruby
# === CONTROLLER SIMULATION ===
# Simulate form submission
simulate_form_submission(handle: 'username', email: 'test@example.com', code: 'abc123')

# === PIPELINE EXECUTION ===
# Run complete pipeline
pipeline = WorkshopPipelineService.new(handle: 'username')
result = pipeline.run

# === DATA VALIDATION ===
# Validate user data quality
user = User.find_by(handle: 'username')
validation = validate_user_data(user)

# === IMAGE PROCESSING ===
# Download and analyze images
avatar_data = download_and_analyze_image(user.avatar_twitter_full_url, 'avatar.jpg')
analysis = analyze_image_with_gemini(avatar_data, :avatar)

# === BACKGROUND JOB ===
# Simulate async processing
WorkshopSubmissionJob.perform_later('handle', 'email', 'code')

# === QUICK ANALYSIS ===
# User overview
user = User.includes(:tweets, :user_stats, :tags).find_by(handle: 'username')
puts "#{user.name}: #{user.followers} followers, #{user.tweets.count} tweets"
puts "Stats: ATK=#{user.attack}, DEF=#{user.defense}, SPD=#{user.speed}"
puts "Tags: #{user.tags.pluck(:name).join(', ')}"
```

### Common Issues and Solutions

```ruby
# Issue: User not found
rescue TwitterApiService::NotFoundError
  puts "User doesn't exist on Twitter"

# Issue: Rate limited
rescue TwitterApiService::RateLimitError
  puts "API rate limited - wait and retry"

# Issue: AI generation failed
if !ai_service.generate!
  puts "Check character limits and try again"

# Issue: Image download failed
if !image_data
  puts "Check image URL and network connectivity"

# Issue: Invalid MIME type
if !['image/jpeg', 'image/png', 'image/webp'].include?(mime_type)
  puts "Unsupported image format for Gemini"
```

This completes Part 1 of the workshop, covering the complete flow from controller submission through data collection and basic analysis. Part 2 will focus on advanced AI analysis, stats calculation improvements, and optimization techniques.
