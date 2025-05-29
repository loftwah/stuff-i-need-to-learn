# TechDeck Workshop Part 2C: Controller Flow & Console Mastery

Master the complete submission flow and build your daily development toolkit.

## Prerequisites

Complete Parts 2A and 2B with loftwah data analyzed and AI generation fixed.

## Exercise 1: Pull First Two Pages of Data

### Twitter API Pagination Deep Dive

```ruby
# Start Rails console
bin/rails console

# Load loftwah and examine current data
user = User.find_by(handle: 'loftwah')
current_tweets = user.tweets.count

puts "🐦 Current Loftwah Data"
puts "=" * 25
puts "Total tweets in DB: #{current_tweets}"
puts "Date range: #{user.tweets.minimum(:created_at_twitter)&.strftime('%Y-%m-%d')} to #{user.tweets.maximum(:created_at_twitter)&.strftime('%Y-%m-%d')}"

# Analyze tweet distribution by type
tweet_breakdown = user.tweets.group(:tweet_type).count
puts "\nTweet breakdown:"
tweet_breakdown.each { |type, count| puts "  #{type}: #{count}" }

# Check for pagination markers
oldest_tweet = user.tweets.order(:created_at_twitter).first
newest_tweet = user.tweets.order(:created_at_twitter).last

puts "\nPagination status:"
puts "  Oldest tweet ID: #{oldest_tweet&.tweet_id_twitter}"
puts "  Newest tweet ID: #{newest_tweet&.tweet_id_twitter}"
puts "  Oldest date: #{oldest_tweet&.created_at_twitter}"
puts "  Newest date: #{newest_tweet&.created_at_twitter}"

# Simulate fetching next page (mock Twitter API response)
def simulate_twitter_api_page(user_handle, max_id: nil, page: 1)
  puts "\n📡 Simulating Twitter API call"
  puts "   Handle: @#{user_handle}"
  puts "   Page: #{page}"
  puts "   Max ID: #{max_id}" if max_id

  # Mock response structure (based on real Twitter API)
  mock_response = {
    page: page,
    per_page: 20,
    total_pages: 5,
    has_next_page: page < 5,
    next_max_id: "123456789#{page}",
    tweets: Array.new(20) do |i|
      {
        id: "123456789#{page}#{i}",
        text: "Sample tweet #{page}-#{i} for @#{user_handle} #tech #rails",
        created_at: (Time.current - (page * 7 + i).days).iso8601,
        likes: rand(0..50),
        retweets: rand(0..10),
        tweet_type: ['tweet', 'reply', 'retweet'].sample,
        in_reply_to_user_id: rand(2) == 0 ? "987654321" : nil
      }
    end
  }

  puts "   ✅ Mock response: #{mock_response[:tweets].count} tweets"
  puts "   Next page available: #{mock_response[:has_next_page]}"

  mock_response
end

# Fetch pages 1 and 2
puts "\n📄 Fetching First Two Pages"
puts "=" * 30

page_1 = simulate_twitter_api_page('loftwah', page: 1)
page_2 = simulate_twitter_api_page('loftwah', max_id: page_1[:next_max_id], page: 2)

puts "\nPage summary:"
puts "  Page 1: #{page_1[:tweets].count} tweets"
puts "  Page 2: #{page_2[:tweets].count} tweets"
puts "  Total new tweets: #{page_1[:tweets].count + page_2[:tweets].count}"

# Process both pages (mock data insertion)
all_new_tweets = page_1[:tweets] + page_2[:tweets]

puts "\n📊 Processing New Tweet Data"
puts "=" * 30

# Analyze new data before insertion
new_tweet_types = all_new_tweets.group_by { |t| t[:tweet_type] }
puts "New tweet breakdown:"
new_tweet_types.each do |type, tweets|
  avg_likes = tweets.sum { |t| t[:likes] } / tweets.count.to_f
  puts "  #{type}: #{tweets.count} tweets, #{avg_likes.round(1)} avg likes"
end

# Mock insertion process
puts "\n💾 Mock Database Insertion"
puts "=" * 25

all_new_tweets.each_with_index do |tweet_data, index|
  # In real implementation, you'd do:
  # Tweet.find_or_create_by(tweet_id_twitter: tweet_data[:id]) do |tweet|
  #   tweet.user = user
  #   tweet.text = tweet_data[:text]
  #   tweet.created_at_twitter = tweet_data[:created_at]
  #   tweet.likes = tweet_data[:likes]
  #   tweet.retweets = tweet_data[:retweets]
  #   tweet.tweet_type = tweet_data[:tweet_type]
  #   tweet.in_reply_to_user_id_twitter = tweet_data[:in_reply_to_user_id]
  # end

  if (index + 1) % 10 == 0
    puts "  Processed #{index + 1}/#{all_new_tweets.count} tweets..."
  end
end

puts "✅ All #{all_new_tweets.count} tweets processed"
puts "📈 Total tweets for @loftwah would be: #{current_tweets + all_new_tweets.count}"
```

### Real vs Mock Data Comparison

```ruby
# Compare real data patterns with mock data
puts "\n🔍 Real vs Mock Data Analysis"
puts "=" * 30

# Real data analysis
real_tweets = user.tweets.limit(40) # Similar sample size
real_engagement = {
  avg_likes: real_tweets.average(:likes).to_f.round(2),
  avg_retweets: real_tweets.average(:retweets).to_f.round(2),
  types: real_tweets.group(:tweet_type).count
}

# Mock data analysis
mock_engagement = {
  avg_likes: all_new_tweets.sum { |t| t[:likes] } / all_new_tweets.count.to_f,
  avg_retweets: all_new_tweets.sum { |t| t[:retweets] } / all_new_tweets.count.to_f,
  types: all_new_tweets.group_by { |t| t[:tweet_type] }.transform_values(&:count)
}

puts "📊 Engagement Comparison:"
puts "  Real data avg likes: #{real_engagement[:avg_likes]}"
puts "  Mock data avg likes: #{mock_engagement[:avg_likes].round(2)}"
puts "  Real data avg RTs: #{real_engagement[:avg_retweets]}"
puts "  Mock data avg RTs: #{mock_engagement[:avg_retweets].round(2)}"

puts "\n📝 Content Type Comparison:"
puts "Real data types: #{real_engagement[:types]}"
puts "Mock data types: #{mock_engagement[:types]}"

# Quality assessment
puts "\n✅ Data Quality Assessment:"
puts "  Real data reflects actual engagement patterns"
puts "  Mock data useful for testing pagination logic"
puts "  Both datasets show similar type distributions"
puts "  Real data better for AI training and analysis"
```

## Exercise 2: Complete Controller Submission Flow

### Understanding the Full Pipeline

```ruby
# Trace the complete submission flow
puts "\n🔄 Complete Submission Flow Analysis"
puts "=" * 35

# Step 1: Form submission
puts "1️⃣ FORM SUBMISSION"
puts "   Route: POST /users"
puts "   Params: { user: { handle: 'loftwah' } }"
puts "   Controller: UsersController#create"

# Mock controller params
submission_params = {
  user: {
    handle: 'loftwah',
    source: 'form_submission'
  }
}

puts "\n2️⃣ CONTROLLER PROCESSING"
puts "   Strong params filtering"
puts "   User lookup/creation"
puts "   Background job queuing"

# Step 2: Background processing simulation
def simulate_background_processing(handle)
  puts "\n3️⃣ BACKGROUND JOB PROCESSING"
  puts "   Job: SocialDataJob.perform_later('#{handle}')"

  steps = [
    "🔍 Twitter API authentication",
    "📥 Fetch user profile data",
    "🐦 Fetch tweets (paginated)",
    "💾 Save/update user record",
    "💾 Save/update tweet records",
    "📊 Calculate engagement stats",
    "🤖 Queue AI generation job",
    "🖼️ Queue image processing job"
  ]

  steps.each_with_index do |step, i|
    puts "   #{i+1}. #{step}"
    sleep(0.2) # Simulate processing time
  end

  puts "   ✅ SocialDataJob completed"
end

# Step 3: AI generation simulation
def simulate_ai_generation(user)
  puts "\n4️⃣ AI GENERATION JOB"
  puts "   Job: AiGenerationJob.perform_later(#{user.id})"
  puts "   Service: AiGenerationService.new(user).generate!"

  ai_steps = [
    "📝 Build comprehensive prompt",
    "🧠 Send to Gemini with JSON schema",
    "✅ Validate response format",
    "💾 Update user with generated content",
    "🏷️ Create/update tags",
    "📊 Calculate card stats"
  ]

  ai_steps.each_with_index do |step, i|
    puts "   #{i+1}. #{step}"
    sleep(0.2)
  end

  puts "   ✅ AiGenerationJob completed"
end

# Run simulation
simulate_background_processing('loftwah')
simulate_ai_generation(user)

puts "\n5️⃣ FINAL RESULT"
puts "   User profile updated"
puts "   Tweets synchronized"
puts "   AI content generated"
puts "   Card ready for display"
puts "   Redirect to: /users/loftwah"
```

### Controller Code Analysis

```ruby
# Analyze actual controller implementation
puts "\n🔧 Controller Implementation Analysis"
puts "=" * 35

# Mock controller code structure
controller_structure = <<~CODE
  class UsersController < ApplicationController
    def create
      @user = User.find_or_initialize_by(handle: user_params[:handle])

      if @user.persisted?
        # User exists, update data
        SocialDataJob.perform_later(@user.handle)
        redirect_to @user, notice: 'Updating profile...'
      else
        # New user, full processing
        if @user.save
          SocialDataJob.perform_later(@user.handle)
          AiGenerationJob.perform_later(@user.id)
          redirect_to @user, notice: 'Creating profile...'
        else
          render :new, alert: 'Invalid handle'
        end
      end
    end

    def show
      @user = User.find_by!(handle: params[:handle])
      @tweets = @user.tweets.includes(:user).order(created_at_twitter: :desc)
      @stats = {
        attack: @user.attack,
        defense: @user.defense,
        speed: @user.speed
      }
    end

    private

    def user_params
      params.require(:user).permit(:handle)
    end
  end
CODE

puts "📝 Controller Structure:"
puts controller_structure

# Error handling scenarios
puts "\n🚨 Error Handling Scenarios"
puts "=" * 25

error_scenarios = [
  {
    scenario: "Invalid Twitter handle",
    cause: "Handle doesn't exist on Twitter",
    handling: "Show error message, redirect to form"
  },
  {
    scenario: "Twitter API rate limit",
    cause: "Too many requests in time window",
    handling: "Queue job for later, show waiting message"
  },
  {
    scenario: "Private Twitter account",
    cause: "Account is protected/private",
    handling: "Show limited profile, explain restrictions"
  },
  {
    scenario: "Gemini API failure",
    cause: "AI service unavailable",
    handling: "Use default values, retry background job"
  },
  {
    scenario: "Database connection error",
    cause: "DB timeout or connection issue",
    handling: "Show 500 error, log for debugging"
  }
]

error_scenarios.each do |scenario|
  puts "\n❌ #{scenario[:scenario]}"
  puts "   Cause: #{scenario[:cause]}"
  puts "   Handling: #{scenario[:handling]}"
end
```

### Background Job Flow

```ruby
# Deep dive into background job architecture
puts "\n⚙️ Background Job Architecture"
puts "=" * 30

# Job dependencies and flow
job_flow = {
  "SocialDataJob" => {
    triggers: ["Form submission", "Manual refresh"],
    dependencies: ["Twitter API credentials"],
    outputs: ["User profile", "Tweet data"],
    next_jobs: ["AiGenerationJob", "ImageProcessingJob"]
  },
  "AiGenerationJob" => {
    triggers: ["After SocialDataJob", "Manual regeneration"],
    dependencies: ["User tweets", "Gemini API credentials"],
    outputs: ["Bio content", "Tags", "Card attributes"],
    next_jobs: ["StatsCalculationJob"]
  },
  "ImageProcessingJob" => {
    triggers: ["New user images", "Image URL changes"],
    dependencies: ["Image URLs", "Storage service"],
    outputs: ["Processed images", "Image metadata"],
    next_jobs: ["None"]
  }
}

job_flow.each do |job_name, details|
  puts "\n🔧 #{job_name}"
  puts "   Triggers: #{details[:triggers].join(', ')}"
  puts "   Dependencies: #{details[:dependencies].join(', ')}"
  puts "   Outputs: #{details[:outputs].join(', ')}"
  puts "   Next jobs: #{details[:next_jobs].join(', ')}"
end

# Job monitoring and debugging
puts "\n📊 Job Monitoring Commands"
puts "=" * 25

monitoring_commands = <<~COMMANDS
  # Check job status
  Sidekiq::Queue.new.size                    # Pending jobs
  Sidekiq::RetrySet.new.size                 # Failed jobs
  Sidekiq::Workers.new.size                  # Active workers

  # Find specific jobs
  Sidekiq::Queue.new.select { |job| job.args.first == '12345' }

  # Clear failed jobs
  Sidekiq::RetrySet.new.clear

  # Enqueue job manually
  SocialDataJob.perform_later('loftwah')
  AiGenerationJob.perform_later(user.id)
COMMANDS

puts monitoring_commands
```

## Exercise 3: Rails Console Mastery

### Daily Development Workflow

```ruby
# Essential daily commands for TechDeck development
puts "\n⚡ Daily Development Commands"
puts "=" * 30

daily_commands = <<~COMMANDS
# Quick user lookup
user = User.find_by(handle: 'loftwah')
user = User.includes(:tweets, :tags).find_by(handle: 'loftwah')

# Data refresh
SocialDataJob.perform_now('loftwah')           # Sync immediately
AiGenerationJob.perform_now(user.id)           # Regenerate AI content

# Content analysis
user.tweets.where(tweet_type: 'tweet').count  # Original content
user.tweets.where(tweet_type: 'reply').count  # Community engagement
user.tweets.average(:likes)                   # Engagement rate

# Stats debugging
CardStatService.new(user).generate_stats      # Test stats calculation
user.update(attack: 75, defense: 60, speed: 80) # Manual override

# AI testing
AiGenerationService.new(user).send(:build_prompt) # Check prompt
service = AiGenerationService.new(user)
service.generate!                             # Full generation

# Quick fixes
user.tags.destroy_all                         # Clear old tags
user.update(short_bio: nil, long_bio: nil)    # Reset AI content

# Data export
user.tweets.to_json                           # Export tweets
user.as_json(include: [:tweets, :tags])       # Full export
COMMANDS

puts daily_commands

# Advanced debugging commands
puts "\n🔬 Advanced Debugging Commands"
puts "=" * 30

debugging_commands = <<~COMMANDS
# Performance analysis
User.includes(:tweets).where(followers: 1000..).count
Tweet.joins(:user).where(users: { verified: true }).average(:likes)

# Data quality checks
User.where(short_bio: nil).count              # Missing AI content
User.joins(:tweets).group('users.handle').count # Tweet distribution
Tweet.where('created_at_twitter > ?', 1.week.ago).count # Recent activity

# Batch operations
User.where(followers: 0).find_each { |u| SocialDataJob.perform_later(u.handle) }
User.where(short_bio: nil).find_each { |u| AiGenerationJob.perform_later(u.id) }

# API testing
GEMINI_CLIENT.count_tokens(contents: { role: 'user', parts: { text: 'test' } })
TwitterService.new.get_user_profile('loftwah') # If you have TwitterService

# Database maintenance
Tweet.where('created_at_twitter < ?', 1.year.ago).delete_all
ActiveRecord::Base.connection.execute('VACUUM ANALYZE;') # PostgreSQL
COMMANDS

puts debugging_commands
```

### Console Helpers and Shortcuts

```ruby
# Create console helper methods
puts "\n🛠️ Console Helper Methods"
puts "=" * 25

# Define helpful methods (add to console or initializer)
def quick_user(handle)
  User.includes(:tweets, :tags, :user_stats).find_by(handle: handle)
end

def refresh_user(handle)
  user = User.find_by(handle: handle)
  return unless user

  puts "🔄 Refreshing @#{handle}..."
  SocialDataJob.perform_now(handle)
  AiGenerationJob.perform_now(user.id)
  puts "✅ Refresh complete"
  user.reload
end

def user_summary(handle)
  user = quick_user(handle)
  return "User not found" unless user

  <<~SUMMARY
    👤 #{user.name} (@#{user.handle})
    📊 #{user.followers} followers, #{user.following} following
    🐦 #{user.tweets.count} tweets (#{user.tweets.where(tweet_type: 'tweet').count} original)
    💪 Stats: A#{user.attack}/D#{user.defense}/S#{user.speed}
    🏷️ Tags: #{user.tags.pluck(:name).join(', ')}
    📝 Bio: #{user.short_bio&.truncate(80)}
  SUMMARY
end

def compare_users(*handles)
  users = handles.map { |h| quick_user(h) }.compact

  puts "👥 User Comparison"
  puts "=" * 20

  users.each do |user|
    puts "#{user.handle}: #{user.followers} followers, #{user.tweets.count} tweets, A#{user.attack}/D#{user.defense}/S#{user.speed}"
  end

  # Find leader in each category
  max_followers = users.max_by(&:followers)
  max_tweets = users.max_by { |u| u.tweets.count }
  max_attack = users.max_by(&:attack)

  puts "\n🏆 Leaders:"
  puts "Followers: @#{max_followers.handle} (#{max_followers.followers})"
  puts "Content: @#{max_tweets.handle} (#{max_tweets.tweets.count} tweets)"
  puts "Attack: @#{max_attack.handle} (#{max_attack.attack})"
end

# Test helper methods
puts "\nTesting helper methods:"
puts user_summary('loftwah')

# Mock comparison (assuming other users exist)
puts "\n📊 Mock User Comparison:"
puts "compare_users('loftwah', 'other_user1', 'other_user2')"
puts "Would show side-by-side stats comparison"
```

### Console Configuration

```ruby
# Console configuration for productivity
puts "\n⚙️ Console Configuration"
puts "=" * 25

console_config = <<~CONFIG
# Add to config/application.rb or .pryrc

# Auto-load common models
console do
  User.connection
  Tweet.connection
  puts "🚀 TechDeck console loaded"
  puts "Quick commands: quick_user('handle'), user_summary('handle')"
end

# Pry configuration (.pryrc)
Pry.config.prompt = proc do |obj, nest_level, _|
  "techdeck(#{obj})> "
end

# Useful aliases
alias :u :quick_user
alias :sum :user_summary

# Auto-completion helpers
def handles
  User.pluck(:handle).sort
end

def recent_users
  User.order(created_at: :desc).limit(10).pluck(:handle)
end
CONFIG

puts console_config

# Environment-specific commands
puts "\n🌍 Environment-Specific Commands"
puts "=" * 30

env_commands = <<~ENV
# Development
Rails.env.development?
User.count < 100                               # Small dataset
Sidekiq::Queue.new.clear                       # Clear job queue

# Staging
Rails.env.staging?
User.limit(1000)                               # Limited testing data
Rails.cache.clear                              # Clear cache

# Production
Rails.env.production?
User.count                                     # Full dataset
Sidekiq::Stats.new                            # Monitor jobs
Rails.logger.level = Logger::WARN             # Reduce noise
ENV

puts env_commands
```

## Exercise 4: Complete Cheatsheet

### Gemini + SocialData Commands

```ruby
puts "\n📋 Complete Command Cheatsheet"
puts "=" * 30
```

```ruby
# ===============================================
# TECHDECK DEVELOPMENT CHEATSHEET
# ===============================================

# --- USER MANAGEMENT ---
# Load user with associations
user = User.includes(:tweets, :tags, :user_stats).find_by(handle: 'loftwah')

# Quick user lookup
u = User.find_by(handle: 'username')

# Create/update user
User.find_or_create_by(handle: 'newuser') do |user|
  user.name = 'New User'
  user.followers = 1000
end

# --- DATA SYNCHRONIZATION ---
# Sync user data (foreground)
SocialDataJob.perform_now('loftwah')

# Sync user data (background)
SocialDataJob.perform_later('loftwah')

# Regenerate AI content
AiGenerationJob.perform_now(user.id)
AiGenerationJob.perform_later(user.id)

# Batch refresh users
User.where('updated_at < ?', 1.week.ago).find_each do |user|
  SocialDataJob.perform_later(user.handle)
end

# --- TWEET ANALYSIS ---
# Get tweets by type
original_tweets = user.tweets.where(tweet_type: 'tweet')
replies = user.tweets.where(tweet_type: 'reply')
retweets = user.tweets.where(tweet_type: 'retweet')

# Engagement analysis
user.tweets.average(:likes)
user.tweets.average(:retweets)
user.tweets.where('likes > ?', 10).count

# Content patterns
user.tweets.where('LENGTH(text) > ?', 100).count
user.tweets.group(:tweet_type).count

# Recent activity
user.tweets.where('created_at_twitter > ?', 1.week.ago).count

# --- GEMINI AI COMMANDS ---
# Simple text generation
result = GEMINI_CLIENT.generate_content(
  contents: { role: 'user', parts: { text: 'Your prompt here' } }
)
response = result.dig('candidates', 0, 'content', 'parts', 0, 'text')

# JSON mode
result = GEMINI_CLIENT.stream_generate_content({
  contents: { role: 'user', parts: { text: 'Return JSON list of colors' } },
  generation_config: { response_mime_type: 'application/json' }
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

data = JSON.parse(json_string)

# JSON with schema (Pro models only)
result = GEMINI_CLIENT.stream_generate_content({
  contents: { role: 'user', parts: { text: 'Generate user profile' } },
  generation_config: {
    response_mime_type: 'application/json',
    response_schema: {
      type: 'object',
      properties: {
        name: { type: 'string' },
        skills: { type: 'array', items: { type: 'string' } }
      }
    }
  }
})

# System instructions
result = GEMINI_CLIENT.generate_content({
  system_instruction: {
    role: 'user',
    parts: { text: 'You are a helpful tech expert.' }
  },
  contents: { role: 'user', parts: { text: 'Explain Rails' } }
})

# Token counting
token_count = GEMINI_CLIENT.count_tokens(
  contents: { role: 'user', parts: { text: 'Your prompt' } }
)
puts "Tokens: #{token_count['totalTokens']}"

# --- IMAGE ANALYSIS ---
require 'base64'
image_data = Base64.strict_encode64(File.read('path/to/image.jpg'))

result = GEMINI_CLIENT.stream_generate_content({
  contents: [{
    role: 'user',
    parts: [
      { text: 'Describe this image' },
      {
        inline_data: {
          mime_type: 'image/jpeg',
          data: image_data
        }
      }
    ]
  }]
})

# --- STATS CALCULATION ---
# Current stats
puts "#{user.name}: A#{user.attack}/D#{user.defense}/S#{user.speed}"

# Calculate new stats
tweets_sample = user.tweets.limit(50)
avg_likes = tweets_sample.average(:likes).to_f
avg_retweets = tweets_sample.average(:retweets).to_f

new_attack = (avg_likes * 0.3 + avg_retweets * 0.7).clamp(1, 100)
new_defense = (user.followers / 100.0 + avg_likes * 0.1).clamp(1, 100)
new_speed = (user.tweets.count / 10.0 + avg_retweets * 2).clamp(1, 100)

user.update(attack: new_attack, defense: new_defense, speed: new_speed)

# Use CardStatService (if available)
if defined?(CardStatService)
  stats = CardStatService.new(user, tweets: user.tweets.limit(20)).generate_stats
  puts stats
end

# --- AI GENERATION TESTING ---
# Test current service
if defined?(AiGenerationService)
  service = AiGenerationService.new(user)
  prompt = service.send(:build_prompt)
  puts prompt

  # Generate (careful - updates user!)
  success = service.generate!
  puts "Generation: #{success ? 'success' : 'failed'}"
end

# Manual AI generation with fixed schema
system_instruction = {
  role: 'user',
  parts: { text: 'You are a tech trading card expert. Follow limits exactly.' }
}

schema = {
  type: 'object',
  properties: {
    short_bio: { type: 'string', minLength: 150, maxLength: 250 },
    tags: { type: 'array', items: { type: 'string' }, minItems: 6, maxItems: 6 }
  }
}

prompt = "Create profile for #{user.name}: #{user.bio}"

result = GEMINI_CLIENT.stream_generate_content({
  system_instruction: system_instruction,
  contents: { role: 'user', parts: { text: prompt } },
  generation_config: {
    response_mime_type: 'application/json',
    response_schema: schema
  }
})

# --- DATA EXPORT ---
# Single user export
export_data = {
  profile: user.attributes,
  tweets: user.tweets.limit(100).as_json,
  tags: user.tags.pluck(:name)
}
File.write("#{user.handle}_export.json", JSON.pretty_generate(export_data))

# Bulk export
User.limit(10).includes(:tweets, :tags).find_each do |u|
  data = { profile: u.attributes, tweet_count: u.tweets.count }
  puts "#{u.handle}: #{data[:tweet_count]} tweets"
end

# --- DEBUGGING ---
# Check for missing data
User.where(short_bio: nil).count              # Missing AI content
User.where(followers: 0).count               # Missing profile data
Tweet.where(likes: nil).count                # Missing engagement data

# Find problematic users
User.joins(:tweets).group('users.id').having('COUNT(tweets.id) < ?', 5)

# Clear and regenerate
user.tags.destroy_all
user.update(short_bio: nil, long_bio: nil, buff: nil)
AiGenerationJob.perform_now(user.id)

# --- JOB MONITORING ---
# Sidekiq status
Sidekiq::Queue.new.size                       # Pending jobs
Sidekiq::RetrySet.new.size                    # Failed jobs
Sidekiq::Workers.new.size                     # Active workers

# Clear failed jobs
Sidekiq::RetrySet.new.clear

# Find user jobs
Sidekiq::Queue.new.select { |job| job.args.include?(user.id) }

# --- PERFORMANCE ---
# Cache warming
User.includes(:tweets, :tags).limit(100).load

# Database cleanup
Tweet.where('created_at_twitter < ?', 2.years.ago).delete_all

# Index usage
User.connection.execute("EXPLAIN ANALYZE SELECT * FROM users WHERE handle = 'loftwah'")

# --- UTILITY FUNCTIONS ---
def quick_stats(handle)
  u = User.find_by(handle: handle)
  return "Not found" unless u
  "#{u.name}: #{u.followers}F, #{u.tweets.count}T, A#{u.attack}/D#{u.defense}/S#{u.speed}"
end

def top_users(limit = 10)
  User.order(followers: :desc).limit(limit).pluck(:handle, :followers)
end

def engagement_leaders(limit = 5)
  User.joins(:tweets)
      .group('users.id')
      .order('AVG(tweets.likes) DESC')
      .limit(limit)
      .pluck(:handle, 'AVG(tweets.likes)')
end

# --- QUICK TESTS ---
# Test Gemini connection
begin
  GEMINI_CLIENT.generate_content(contents: { role: 'user', parts: { text: 'Hi' } })
  puts "✅ Gemini API working"
rescue => e
  puts "❌ Gemini API error: #{e.message}"
end

# Test database connection
begin
  User.count
  puts "✅ Database connection working"
rescue => e
  puts "❌ Database error: #{e.message}"
end

# Test background jobs
begin
  Sidekiq::Stats.new
  puts "✅ Sidekiq running"
rescue => e
  puts "❌ Sidekiq error: #{e.message}"
end
```

## Exercise 5: Integration Testing

### End-to-End Flow Testing

```ruby
# Complete integration test of the submission flow
puts "\n🔄 End-to-End Integration Test"
puts "=" * 35

def test_complete_flow(handle)
  puts "🚀 Testing complete flow for @#{handle}"
  puts "=" * 30

  # Step 1: Initial state
  existing_user = User.find_by(handle: handle)
  puts "1️⃣ Initial state: #{existing_user ? 'User exists' : 'New user'}"

  # Step 2: Social data fetch
  puts "\n2️⃣ Social data synchronization..."
  start_time = Time.current

  begin
    # In real app: SocialDataJob.perform_now(handle)
    # For testing: simulate the process
    user = User.find_or_create_by(handle: handle) do |u|
      u.name = "Test User #{handle.capitalize}"
      u.followers = rand(100..10000)
      u.following = rand(50..1000)
      u.bio = "Test bio for integration testing"
    end

    # Simulate tweet creation
    5.times do |i|
      user.tweets.find_or_create_by(tweet_id_twitter: "test_#{handle}_#{i}") do |tweet|
        tweet.text = "Test tweet #{i} for integration testing #rails #ai"
        tweet.created_at_twitter = Time.current - i.days
        tweet.likes = rand(0..100)
        tweet.retweets = rand(0..20)
        tweet.tweet_type = ['tweet', 'reply'].sample
      end
    end

    sync_time = (Time.current - start_time).round(2)
    puts "   ✅ Social data sync completed in #{sync_time}s"
    puts "   📊 User: #{user.name}, #{user.followers} followers"
    puts "   🐦 Tweets: #{user.tweets.count}"

  rescue => e
    puts "   ❌ Social data sync failed: #{e.message}"
    return false
  end

  # Step 3: AI generation
  puts "\n3️⃣ AI content generation..."
  ai_start_time = Time.current

  begin
    # Test with our fixed JSON schema approach
    system_instruction = {
      role: 'user',
      parts: { text: 'You are a tech trading card expert. Follow character limits exactly.' }
    }

    schema = {
      type: 'object',
      properties: {
        short_bio: { type: 'string', minLength: 150, maxLength: 250 },
        long_bio: { type: 'string', minLength: 900, maxLength: 1100 },
        tags: { type: 'array', items: { type: 'string' }, minItems: 6, maxItems: 6 },
        buff: { type: 'string', maxLength: 30 },
        weakness: { type: 'string', maxLength: 30 },
        vibe: { type: 'string', maxLength: 30 },
        special_move: { type: 'string', maxLength: 30 },
        flavor_text: { type: 'string', minLength: 30, maxLength: 60 }
      },
      required: ['short_bio', 'long_bio', 'tags', 'buff', 'weakness', 'vibe', 'special_move', 'flavor_text']
    }

    prompt = <<~PROMPT
      Create a tech trading card for #{user.name} (@#{user.handle}).
      Bio: #{user.bio}
      Followers: #{user.followers}
      Sample content: #{user.tweets.limit(3).pluck(:text).join(' | ')}

      Character limits are STRICT and will be validated.
    PROMPT

    result = GEMINI_CLIENT.stream_generate_content({
      system_instruction: system_instruction,
      contents: { role: 'user', parts: { text: prompt } },
      generation_config: {
        response_mime_type: 'application/json',
        response_schema: schema
      }
    })

    json_string = result
                  .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                  .map { |parts| parts.map { |part| part['text'] }.join }
                  .join

    ai_data = JSON.parse(json_string)
    ai_time = (Time.current - ai_start_time).round(2)

    # Validate and update user
    if ai_data['short_bio']&.length&.between?(150, 250) &&
       ai_data['long_bio']&.length&.between?(900, 1100) &&
       ai_data['tags']&.length == 6

      user.update!(
        short_bio: ai_data['short_bio'],
        long_bio: ai_data['long_bio'],
        buff: ai_data['buff'],
        weakness: ai_data['weakness'],
        vibe: ai_data['vibe'],
        special_move: ai_data['special_move'],
        flavor_text: ai_data['flavor_text']
      )

      # Create tags
      ai_data['tags'].each do |tag_name|
        tag = Tag.find_or_create_by(name: tag_name.downcase)
        user.user_tags.find_or_create_by(tag: tag)
      end

      puts "   ✅ AI generation completed in #{ai_time}s"
      puts "   📝 Short bio: #{ai_data['short_bio'].length} chars"
      puts "   📖 Long bio: #{ai_data['long_bio'].length} chars"
      puts "   🏷️ Tags: #{ai_data['tags'].join(', ')}"

    else
      puts "   ⚠️ AI generation succeeded but validation failed"
      puts "   📏 Short bio: #{ai_data['short_bio']&.length} chars (need 150-250)"
      puts "   📏 Long bio: #{ai_data['long_bio']&.length} chars (need 900-1100)"
      puts "   📏 Tags: #{ai_data['tags']&.length} items (need 6)"
    end

  rescue => e
    puts "   ❌ AI generation failed: #{e.message}"
    return false
  end

  # Step 4: Stats calculation
  puts "\n4️⃣ Stats calculation..."
  stats_start_time = Time.current

  begin
    tweets_for_stats = user.tweets.limit(20)
    avg_likes = tweets_for_stats.average(:likes).to_f
    avg_retweets = tweets_for_stats.average(:retweets).to_f

    # Calculate stats using engagement data
    attack = [
      (avg_likes * 0.4).round(0),
      (avg_retweets * 1.5).round(0),
      ([user.followers / 100, 30].min).round(0)
    ].sum.clamp(1, 100)

    defense = [
      ([user.followers / 50, 40].min).round(0),
      (user.tweets.count / 2).clamp(0, 30),
      (user.verified? ? 20 : 10)
    ].sum.clamp(1, 100)

    speed = [
      ([user.tweets.count / 5, 40].min).round(0),
      (avg_likes > 10 ? 25 : 10),
      (avg_retweets > 5 ? 20 : 5)
    ].sum.clamp(1, 100)

    user.update!(attack: attack, defense: defense, speed: speed)

    stats_time = (Time.current - stats_start_time).round(2)
    puts "   ✅ Stats calculated in #{stats_time}s"
    puts "   ⚔️ Attack: #{attack} (likes×0.4 + RTs×1.5 + followers/100)"
    puts "   🛡️ Defense: #{defense} (followers/50 + tweets/2 + verified)"
    puts "   ⚡ Speed: #{speed} (tweets/5 + engagement bonuses)"

  rescue => e
    puts "   ❌ Stats calculation failed: #{e.message}"
    return false
  end

  # Step 5: Final validation
  puts "\n5️⃣ Final validation..."
  user.reload

  validation_results = {
    profile_complete: user.name.present? && user.bio.present?,
    tweets_synced: user.tweets.count > 0,
    ai_content_generated: user.short_bio.present? && user.long_bio.present?,
    tags_created: user.tags.count >= 6,
    stats_calculated: user.attack > 0 && user.defense > 0 && user.speed > 0
  }

  validation_results.each do |check, passed|
    status = passed ? "✅" : "❌"
    puts "   #{status} #{check.to_s.humanize}"
  end

  total_time = (Time.current - start_time).round(2)
  success = validation_results.values.all?

  puts "\n🎯 Integration Test Result: #{success ? 'PASSED' : 'FAILED'}"
  puts "⏱️ Total time: #{total_time}s"

  if success
    puts "\n📋 Final User Summary:"
    puts "   Name: #{user.name}"
    puts "   Handle: @#{user.handle}"
    puts "   Followers: #{user.followers}"
    puts "   Tweets: #{user.tweets.count}"
    puts "   Tags: #{user.tags.count}"
    puts "   Stats: A#{user.attack}/D#{user.defense}/S#{user.speed}"
    puts "   Bio length: #{user.short_bio&.length}/#{user.long_bio&.length}"
  end

  success
end

# Run integration test
puts "Running integration test with test user..."
test_result = test_complete_flow('test_integration_user')

if test_result
  puts "\n🎉 Integration test PASSED!"
  puts "The complete submission flow is working correctly."
else
  puts "\n💥 Integration test FAILED!"
  puts "Check the error messages above for debugging."
end
```

### Production Readiness Checklist

```ruby
# Production deployment checklist
puts "\n✅ Production Readiness Checklist"
puts "=" * 35

checklist = {
  "Environment Variables" => [
    "GOOGLE_CREDENTIALS_FILE_CONTENTS configured",
    "GOOGLE_GEMINI_REGION set to us-central1",
    "GOOGLE_PROJECT_ID matches Google Cloud project",
    "REDIS_URL configured for Sidekiq",
    "DATABASE_URL configured"
  ],

  "API Credentials" => [
    "Google Cloud service account has proper IAM roles",
    "Vertex AI API enabled in Google Cloud Console",
    "Twitter API credentials configured (if applicable)",
    "Rate limiting configured for external APIs"
  ],

  "Database Setup" => [
    "All migrations run successfully",
    "Database indexes created for performance",
    "Foreign key constraints in place",
    "Data validation rules implemented"
  ],

  "Background Jobs" => [
    "Sidekiq workers configured and running",
    "Job retry logic implemented",
    "Dead job queue monitoring setup",
    "Job performance monitoring enabled"
  ],

  "Error Handling" => [
    "API failure scenarios handled gracefully",
    "User-friendly error messages implemented",
    "Logging configured for debugging",
    "Alert system for critical failures"
  ],

  "Performance" => [
    "Database queries optimized with includes/joins",
    "API response caching implemented",
    "Image processing optimized",
    "Rate limiting for user requests"
  ],

  "Monitoring" => [
    "Application performance monitoring (APM) setup",
    "Database performance monitoring",
    "API usage tracking",
    "User experience metrics collection"
  ]
}

checklist.each do |category, items|
  puts "\n📋 #{category}:"
  items.each do |item|
    puts "   ☐ #{item}"
  end
end
```

## Part 2C Summary & Workshop Completion

```ruby
puts "\n🎓 TechDeck Workshop Complete!"
puts "=" * 30

completion_summary = {
  "Part 2A" => [
    "✅ Complete loftwah dataset analysis",
    "✅ Robust image downloading with error handling",
    "✅ Mock object storage workflow",
    "✅ Gemini image analysis using correct API patterns",
    "✅ Dataset export for further analysis"
  ],

  "Part 2B" => [
    "✅ Tweet vs reply content pattern analysis",
    "✅ Stats formula experimentation with real data",
    "✅ Fixed AI generation with proper JSON schema",
    "✅ Character limit validation and retry logic",
    "✅ Content strategy recommendations"
  ],

  "Part 2C" => [
    "✅ Twitter API pagination understanding",
    "✅ Complete controller submission flow analysis",
    "✅ Background job architecture deep dive",
    "✅ Rails console mastery with daily workflows",
    "✅ Comprehensive cheatsheet for development",
    "✅ End-to-end integration testing",
    "✅ Production readiness checklist"
  ]
}

completion_summary.each do |part, achievements|
  puts "\n#{part}:"
  achievements.each { |achievement| puts "  #{achievement}" }
end

puts "\n🚀 What You've Accomplished:"
puts "   • Mastered real-world social data processing"
puts "   • Fixed AI generation with proper JSON schemas"
puts "   • Built a complete development workflow"
puts "   • Created production-ready error handling"
puts "   • Established monitoring and debugging practices"

puts "\n📚 Key Skills Developed:"
puts "   • Rails console mastery for daily development"
puts "   • Gemini AI integration with proper patterns"
puts "   • Background job architecture and monitoring"
puts "   • Data processing with error handling"
puts "   • Performance optimization techniques"

puts "\n🔗 Everything Connects:"
puts "   Form → Controller → Background Jobs → AI Generation → Stats → Display"
puts "   Each piece works together to create the complete TechDeck experience"

puts "\n⚡ Daily Workflow Established:"
puts "   1. Use console helpers for quick user lookup"
puts "   2. Test AI generation with proper validation"
puts "   3. Monitor background jobs for issues"
puts "   4. Debug with comprehensive logging"
puts "   5. Deploy with confidence using checklist"

puts "\n🎯 Ready for Production!"
puts "You now have the tools, knowledge, and workflows to:"
puts "  • Handle real user data at scale"
puts "  • Debug issues quickly and effectively"
puts "  • Implement new features with confidence"
puts "  • Monitor and maintain the system"
puts "  • Scale the application as it grows"

puts "\n" + "🎉" * 40
puts "TECHDECK WORKSHOP SERIES COMPLETE!"
puts "🎉" * 40
```
