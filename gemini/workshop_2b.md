# TechDeck Workshop Part 2B: AI Content Generation & Stats

Analyze tweet content patterns, fix AI generation, and improve stats calculations using real loftwah data.

## Prerequisites

Complete Part 2A with loftwah's images processed and dataset exported.

## Exercise 1: Tweet vs Reply Content Analysis

### Understanding Content Patterns for AI Prompts

```ruby
# Start Rails console
bin/rails console

# Load loftwah with fresh data
user = User.includes(:tweets, :tags).find_by(handle: 'loftwah')

# Separate content types for analysis
original_tweets = user.tweets.where(tweet_type: 'tweet').order(created_at_twitter: :desc)
replies = user.tweets.where(tweet_type: 'reply').order(created_at_twitter: :desc)
retweets = user.tweets.where(tweet_type: 'retweet').order(created_at_twitter: :desc)

puts "🔍 Content Pattern Analysis"
puts "=" * 30
puts "Original tweets: #{original_tweets.count}"
puts "Replies: #{replies.count}"
puts "Retweets: #{retweets.count}"

# Analyze content differences
puts "\n📝 Content Characteristics:"

if original_tweets.any?
  avg_orig_length = original_tweets.average('LENGTH(text)').to_f.round(0)
  puts "Original tweets avg length: #{avg_orig_length} chars"

  # Sample original tweets
  puts "\n🐦 Original Tweet Samples:"
  original_tweets.limit(5).each_with_index do |tweet, i|
    puts "  #{i+1}. [#{tweet.likes}❤️ #{tweet.retweets}🔁] #{tweet.text.truncate(100)}"
  end
end

if replies.any?
  avg_reply_length = replies.average('LENGTH(text)').to_f.round(0)
  puts "\nReplies avg length: #{avg_reply_length} chars"

  # Sample replies
  puts "\n💬 Reply Samples:"
  replies.limit(5).each_with_index do |tweet, i|
    puts "  #{i+1}. [#{tweet.likes}❤️] #{tweet.text.truncate(100)}"
  end
end

# Content tone analysis with Gemini
puts "\n🤖 AI Content Analysis"
puts "=" * 25

# Analyze original tweets
if original_tweets.any?
  orig_sample = original_tweets.limit(10).pluck(:text).join(' | ')

  orig_prompt = <<~PROMPT
    Analyze these original tweets from @#{user.handle}:

    #{orig_sample}

    Describe:
    1. Writing style and tone
    2. Common themes or topics
    3. Audience engagement approach
    4. Technical vs personal content ratio

    Keep response under 150 words.
  PROMPT

  result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: orig_prompt } }
  })

  orig_analysis = result
                  .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                  .map { |parts| parts.map { |part| part['text'] }.join }
                  .join

  puts "\n📊 Original Tweets Analysis:"
  puts orig_analysis
end

# Analyze replies
if replies.any?
  reply_sample = replies.limit(10).pluck(:text).join(' | ')

  reply_prompt = <<~PROMPT
    Analyze these reply tweets from @#{user.handle}:

    #{reply_sample}

    Describe:
    1. How reply style differs from original tweets
    2. Engagement and interaction patterns
    3. Helpfulness vs promotional content
    4. Community building approach

    Keep response under 150 words.
  PROMPT

  result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: reply_prompt } }
  })

  reply_analysis = result
                   .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                   .map { |parts| parts.map { |part| part['text'] }.join }
                   .join

  puts "\n💬 Replies Analysis:"
  puts reply_analysis
end
```

### Why Different Content Types Need Different Prompts

```ruby
# Demonstrate prompt strategy differences
puts "\n🎯 Prompt Strategy Analysis"
puts "=" * 30

# Strategy 1: Original tweets (broad personality analysis)
orig_strategy = <<~STRATEGY
  Original tweets reveal:
  - Core expertise and interests
  - Personal brand messaging
  - Thought leadership topics
  - Professional positioning

  → Use for: Overall personality, technical skills, industry focus
  → Prompt style: Broad analysis, professional context
STRATEGY

puts "📈 Original Tweet Strategy:"
puts orig_strategy

# Strategy 2: Replies (interaction style analysis)
reply_strategy = <<~STRATEGY
  Replies reveal:
  - Communication style with others
  - Helpfulness and community engagement
  - Technical problem-solving approach
  - Personality in conversations

  → Use for: Social skills, collaboration style, community involvement
  → Prompt style: Interaction focus, relationship building
STRATEGY

puts "\n💬 Reply Analysis Strategy:"
puts reply_strategy

# Test both strategies with loftwah data
puts "\n🧪 Testing Both Strategies"
puts "=" * 25

# Test 1: Broad personality from original tweets
if original_tweets.any?
  broad_prompt = <<~PROMPT
    Based on these original tweets, what are #{user.name}'s core technical interests and professional positioning?

    #{original_tweets.limit(8).pluck(:text).join(' | ')}

    Focus on: expertise areas, industry involvement, thought leadership style
  PROMPT

  result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: broad_prompt } }
  })

  broad_result = result
                 .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                 .map { |parts| parts.map { |part| part['text'] }.join }
                 .join

  puts "🎯 Broad Analysis Result:"
  puts broad_result.truncate(200)
end

# Test 2: Interaction style from replies
if replies.any?
  interaction_prompt = <<~PROMPT
    Based on these reply interactions, how does #{user.name} engage with the tech community?

    #{replies.limit(8).pluck(:text).join(' | ')}

    Focus on: helpfulness, collaboration, community building, communication style
  PROMPT

  result = GEMINI_CLIENT.stream_generate_content({
    contents: { role: 'user', parts: { text: interaction_prompt } }
  })

  interaction_result = result
                       .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                       .map { |parts| parts.map { |part| part['text'] }.join }
                       .join

  puts "\n🤝 Interaction Analysis Result:"
  puts interaction_result.truncate(200)
end
```

## Exercise 2: Card Stats Formula Experimentation

### Current Stats vs Loftwah's Real Data

```ruby
# Analyze current stats calculation
puts "\n📊 Current Stats Analysis"
puts "=" * 25

# Get current stats
current_stats = {
  attack: user.attack,
  defense: user.defense,
  speed: user.speed
}

puts "Current stats for #{user.name}:"
puts "  Attack: #{current_stats[:attack]}"
puts "  Defense: #{current_stats[:defense]}"
puts "  Speed: #{current_stats[:speed]}"

# Calculate new stats with real engagement data
tweets_for_stats = user.tweets.limit(50) # Use recent 50 tweets

engagement_data = {
  avg_likes: tweets_for_stats.average(:likes).to_f.round(2),
  avg_retweets: tweets_for_stats.average(:retweets).to_f.round(2),
  total_tweets: user.tweets.count,
  followers: user.followers,
  following: user.following,
  follower_ratio: (user.followers.to_f / [user.following, 1].max).round(2),
  best_tweet_likes: tweets_for_stats.maximum(:likes) || 0,
  consistency: tweets_for_stats.count > 10 ? (tweets_for_stats.where('likes > ?', tweets_for_stats.average(:likes)).count.to_f / tweets_for_stats.count * 100).round(1) : 0
}

puts "\n📈 Real Engagement Data:"
engagement_data.each do |key, value|
  puts "  #{key}: #{value}"
end

# Test improved stats formulas
puts "\n🧪 Testing Improved Stats Formulas"
puts "=" * 35

# Formula 1: Engagement-weighted
engagement_attack = [
  (engagement_data[:avg_likes] * 0.3).round(0),
  (engagement_data[:avg_retweets] * 0.7).round(0),
  (engagement_data[:best_tweet_likes] * 0.1).round(0)
].sum.clamp(1, 100)

engagement_defense = [
  (engagement_data[:follower_ratio] * 10).round(0),
  (engagement_data[:consistency] * 0.8).round(0),
  ([user.followers / 1000, 50].min).round(0)
].sum.clamp(1, 100)

engagement_speed = [
  ([engagement_data[:total_tweets] / 10, 50].min).round(0),
  (engagement_data[:avg_likes] > 10 ? 30 : 10),
  (user.tweets.where('created_at_twitter > ?', 30.days.ago).count * 2)
].sum.clamp(1, 100)

puts "💪 Engagement-Based Formula:"
puts "  Attack: #{engagement_attack} (avg_likes×0.3 + avg_RTs×0.7 + best×0.1)"
puts "  Defense: #{engagement_defense} (follower_ratio×10 + consistency×0.8 + followers/1000)"
puts "  Speed: #{engagement_speed} (total_tweets/10 + engagement_bonus + recent_activity×2)"

# Formula 2: Influence-focused
influence_attack = [
  ([user.followers / 100, 60].min).round(0),
  (engagement_data[:avg_retweets] * 2).round(0),
  (user.verified? ? 20 : 0)
].sum.clamp(1, 100)

influence_defense = [
  (engagement_data[:follower_ratio] * 15).round(0),
  ([engagement_data[:total_tweets] / 20, 30].min).round(0),
  (engagement_data[:consistency] > 50 ? 25 : 10)
].sum.clamp(1, 100)

influence_speed = [
  (engagement_data[:avg_likes] * 0.5).round(0),
  ([user.tweets.where('created_at_twitter > ?', 7.days.ago).count * 5, 40].min),
  (engagement_data[:avg_retweets] > 5 ? 20 : 5)
].sum.clamp(1, 100)

puts "\n🎯 Influence-Based Formula:"
puts "  Attack: #{influence_attack} (followers/100 + avg_RTs×2 + verified_bonus)"
puts "  Defense: #{influence_defense} (follower_ratio×15 + tweets/20 + consistency_bonus)"
puts "  Speed: #{influence_speed} (avg_likes×0.5 + recent_tweets×5 + RT_bonus)"

# Compare all formulas
puts "\n⚖️ Formula Comparison:"
puts "Current:     A#{current_stats[:attack]}/D#{current_stats[:defense]}/S#{current_stats[:speed]} = #{current_stats.values.sum}"
puts "Engagement:  A#{engagement_attack}/D#{engagement_defense}/S#{engagement_speed} = #{engagement_attack + engagement_defense + engagement_speed}"
puts "Influence:   A#{influence_attack}/D#{influence_defense}/S#{influence_speed} = #{influence_attack + influence_defense + influence_speed}"

# Test with CardStatService for comparison
if defined?(CardStatService)
  puts "\n🔬 CardStatService Comparison:"
  service_stats = CardStatService.new(user, tweets: user.tweets.limit(20)).generate_stats
  puts "Service result: #{service_stats}"
end
```

### AI-Suggested Stats Formula

```ruby
# Get AI suggestions for stats calculation
puts "\n🤖 AI-Suggested Stats Formula"
puts "=" * 30

stats_prompt = <<~PROMPT
  Design RPG-style stats for this tech professional:

  Profile: #{user.name} (@#{user.handle})
  Bio: #{user.bio}
  Followers: #{user.followers}
  Following: #{user.following}

  Engagement Data:
  - Average likes: #{engagement_data[:avg_likes]}
  - Average retweets: #{engagement_data[:avg_retweets]}
  - Total tweets: #{engagement_data[:total_tweets]}
  - Best tweet: #{engagement_data[:best_tweet_likes]} likes
  - Consistency: #{engagement_data[:consistency]}% above average
  - Follower ratio: #{engagement_data[:follower_ratio]}

  Create formulas for Attack, Defense, Speed (1-100 scale) that reflect:
  - Attack: Content impact and reach
  - Defense: Community trust and stability
  - Speed: Activity and responsiveness

  Provide specific calculation formulas using the data above.
PROMPT

result = GEMINI_CLIENT.stream_generate_content({
  contents: { role: 'user', parts: { text: stats_prompt } }
})

ai_stats_suggestion = result
                      .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                      .map { |parts| parts.map { |part| part['text'] }.join }
                      .join

puts ai_stats_suggestion

# Test AI suggestions by extracting numbers (simplified)
puts "\n🧮 Implementing AI Suggestions:"
puts "(Manual implementation based on AI response above)"
# Note: In real implementation, you'd parse the AI response for specific formulas
```

## Exercise 3: Fix AI Prompt Generation

### Test Current AiGenerationService Issues

```ruby
# Test current AI generation with loftwah
puts "\n🚨 Testing Current AI Generation Issues"
puts "=" * 40

if defined?(AiGenerationService)
  puts "Testing AiGenerationService with loftwah..."

  # Check current prompt
  service = AiGenerationService.new(user)
  current_prompt = service.send(:build_prompt) rescue "Error accessing build_prompt method"

  puts "\n📝 Current Prompt (first 300 chars):"
  puts current_prompt.to_s.truncate(300)

  # Test generation (be careful - this might update the user!)
  puts "\n⚠️ WARNING: This will attempt to update the user record!"
  puts "Continue? (y/N)"

  # Uncomment to test (BE CAREFUL):
  # response = gets.chomp.downcase
  # if response == 'y'
  #   success = service.generate!
  #   puts "Generation result: #{success}"
  #
  #   if success
  #     user.reload
  #     puts "Generated short bio: #{user.short_bio&.length} chars"
  #     puts "Generated long bio: #{user.long_bio&.length} chars"
  #     puts "Generated tags: #{user.tags.pluck(:name)}"
  #   end
  # end

  puts "Skipping actual generation for safety. Remove comments to test."
else
  puts "❌ AiGenerationService not found"
end
```

### Fixed JSON Schema Generation

```ruby
# Implement proper JSON schema generation using your fixed patterns
puts "\n✅ Fixed AI Generation with JSON Schema"
puts "=" * 40

# System instruction for better results
system_instruction = {
  role: 'user',
  parts: {
    text: <<~INSTRUCTION
      You are an expert at creating tech trading card profiles.
      You must follow character limits EXACTLY or your output will be rejected.
      Always respond in valid JSON format with no additional text.
      Third-person perspective for all biographical content.
    INSTRUCTION
  }
}

# Proper schema following your documentation
user_card_schema = {
  type: 'object',
  properties: {
    short_bio: {
      type: 'string',
      minLength: 150,
      maxLength: 250,
      description: 'Third-person bio, exactly 150-250 characters'
    },
    long_bio: {
      type: 'string',
      minLength: 900,
      maxLength: 1100,
      description: 'Detailed third-person bio, exactly 900-1100 characters'
    },
    tags: {
      type: 'array',
      items: { type: 'string' },
      minItems: 6,
      maxItems: 6,
      description: 'Exactly 6 lowercase single-word technical skills'
    },
    buff: {
      type: 'string',
      maxLength: 30,
      description: 'Core technical strength, max 30 characters'
    },
    weakness: {
      type: 'string',
      maxLength: 30,
      description: 'Professional challenge area, max 30 characters'
    },
    vibe: {
      type: 'string',
      maxLength: 30,
      description: 'Personality descriptor, max 30 characters'
    },
    special_move: {
      type: 'string',
      maxLength: 30,
      description: 'Signature technical ability, max 30 characters'
    },
    flavor_text: {
      type: 'string',
      minLength: 30,
      maxLength: 60,
      description: 'Memorable quote or motto, 30-60 characters'
    }
  },
  required: ['short_bio', 'long_bio', 'tags', 'buff', 'weakness', 'vibe', 'special_move', 'flavor_text']
}

# Build comprehensive prompt with real data
def build_fixed_prompt(user, original_tweets, replies)
  tweet_sample = original_tweets.limit(8).pluck(:text).join(' | ')
  reply_sample = replies.limit(5).pluck(:text).join(' | ')

  <<~PROMPT
    Create a tech trading card profile for #{user.name} (@#{user.handle}).

    PROFILE DATA:
    - Bio: #{user.bio}
    - Location: #{user.location}
    - Followers: #{user.followers}
    - Following: #{user.following}
    - Verified: #{user.verified?}

    CONTENT ANALYSIS:
    Original tweets (shows expertise): #{tweet_sample.truncate(400)}

    Reply interactions (shows personality): #{reply_sample.truncate(300)}

    CRITICAL REQUIREMENTS:
    - short_bio: EXACTLY 150-250 characters, third-person
    - long_bio: EXACTLY 900-1100 characters, third-person with newlines
    - tags: EXACTLY 6 single lowercase words
    - All other fields: Under 30 characters
    - flavor_text: 30-60 characters
    - Professional yet engaging tone
    - Reflect actual technical expertise and personality
  PROMPT
end

# Generate with proper error handling
def generate_user_card_fixed(user, original_tweets, replies)
  prompt = build_fixed_prompt(user, original_tweets, replies)

  puts "🤖 Generating with fixed JSON schema..."
  puts "Prompt length: #{prompt.length} characters"

  begin
    # Use stream_generate_content exactly as in your docs
    result = GEMINI_CLIENT.stream_generate_content({
      system_instruction: system_instruction,
      contents: {
        role: 'user',
        parts: { text: prompt }
      },
      generation_config: {
        response_mime_type: 'application/json',
        response_schema: user_card_schema
      }
    })

    # Process streaming response exactly as in your docs
    json_string = result
                  .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                  .map { |parts| parts.map { |part| part['text'] }.join }
                  .join

    # Parse and validate
    data = JSON.parse(json_string)
    puts "✅ JSON generation successful"

    # Validate character limits
    validation_errors = []

    if data['short_bio']&.length&.outside?(150..250)
      validation_errors << "short_bio length: #{data['short_bio']&.length} (need 150-250)"
    end

    if data['long_bio']&.length&.outside?(900..1100)
      validation_errors << "long_bio length: #{data['long_bio']&.length} (need 900-1100)"
    end

    if data['tags']&.length != 6
      validation_errors << "tags count: #{data['tags']&.length} (need exactly 6)"
    end

    ['buff', 'weakness', 'vibe', 'special_move'].each do |field|
      if data[field]&.length.to_i > 30
        validation_errors << "#{field} length: #{data[field]&.length} (max 30)"
      end
    end

    if data['flavor_text']&.length&.outside?(30..60)
      validation_errors << "flavor_text length: #{data['flavor_text']&.length} (need 30-60)"
    end

    if validation_errors.any?
      puts "⚠️ Validation issues:"
      validation_errors.each { |error| puts "  - #{error}" }
    else
      puts "✅ All validations passed"
    end

    data

  rescue JSON::ParserError => e
    puts "❌ JSON parsing failed: #{e.message}"
    nil
  rescue => e
    puts "❌ Generation failed: #{e.message}"
    nil
  end
end

# Test fixed generation
puts "\n🧪 Testing Fixed Generation"
puts "=" * 25

if original_tweets.any? && replies.any?
  result = generate_user_card_fixed(user, original_tweets, replies)

  if result
    puts "\n📊 Generated Content:"
    puts "Short bio (#{result['short_bio']&.length}): #{result['short_bio']&.truncate(100)}..."
    puts "Long bio (#{result['long_bio']&.length}): #{result['long_bio']&.truncate(100)}..."
    puts "Tags: #{result['tags']&.join(', ')}"
    puts "Buff: #{result['buff']}"
    puts "Weakness: #{result['weakness']}"
    puts "Vibe: #{result['vibe']}"
    puts "Special move: #{result['special_move']}"
    puts "Flavor text: #{result['flavor_text']}"

    # Store result for comparison
    @fixed_generation_result = result
  end
else
  puts "❌ Need both original tweets and replies for testing"
end
```

### Character Limit Validation & Retry Logic

```ruby
# Implement retry logic for character limit failures
puts "\n🔄 Retry Logic for Character Limits"
puts "=" * 35

def generate_with_retry(user, original_tweets, replies, max_attempts: 3)
  attempts = 0

  while attempts < max_attempts
    attempts += 1
    puts "\n🔄 Attempt #{attempts}/#{max_attempts}"

    result = generate_user_card_fixed(user, original_tweets, replies)
    return nil unless result

    # Check critical limits
    short_bio_valid = result['short_bio']&.length&.between?(150, 250)
    long_bio_valid = result['long_bio']&.length&.between?(900, 1100)
    tags_valid = result['tags']&.length == 6

    if short_bio_valid && long_bio_valid && tags_valid
      puts "✅ Attempt #{attempts} succeeded with valid limits"
      return result
    else
      puts "❌ Attempt #{attempts} failed validation"
      puts "  Short bio: #{result['short_bio']&.length} chars (need 150-250)"
      puts "  Long bio: #{result['long_bio']&.length} chars (need 900-1100)"
      puts "  Tags: #{result['tags']&.length} items (need 6)"

      if attempts < max_attempts
        puts "  Retrying with adjusted prompt..."
        sleep(1) # Rate limiting
      end
    end
  end

  puts "❌ All #{max_attempts} attempts failed validation"
  nil
end

# Test retry logic
if original_tweets.any? && replies.any?
  puts "Testing retry logic (this may take a few minutes)..."

  # Uncomment to test retry logic:
  # final_result = generate_with_retry(user, original_tweets, replies, max_attempts: 2)
  #
  # if final_result
  #   puts "\n🎉 Final successful generation:"
  #   puts "All character limits met!"
  # else
  #   puts "\n😞 Could not generate valid content within retry limit"
  # end

  puts "Retry testing skipped. Remove comments to test."
end
```

## Exercise 4: Content Strategy Recommendations

### AI-Powered Content Strategy

```ruby
# Generate content strategy based on analysis
puts "\n🎯 AI-Powered Content Strategy"
puts "=" * 30

strategy_prompt = <<~PROMPT
  Based on this comprehensive analysis of @#{user.handle}, recommend improvements:

  CURRENT PERFORMANCE:
  - Followers: #{user.followers}
  - Avg likes: #{engagement_data[:avg_likes]}
  - Avg retweets: #{engagement_data[:avg_retweets]}
  - Engagement consistency: #{engagement_data[:consistency]}%
  - Follower ratio: #{engagement_data[:follower_ratio]}

  CONTENT ANALYSIS:
  - Original tweets: #{original_tweets.count} (main content)
  - Replies: #{replies.count} (community engagement)
  - Best tweet performance: #{engagement_data[:best_tweet_likes]} likes

  CURRENT BIO: #{user.bio}

  Provide specific recommendations for:
  1. Content strategy improvements
  2. Engagement optimization
  3. Technical brand positioning
  4. Community building approach

  Keep response actionable and under 300 words.
PROMPT

result = GEMINI_CLIENT.stream_generate_content({
  contents: { role: 'user', parts: { text: strategy_prompt } }
})

strategy_recommendations = result
                          .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                          .map { |parts| parts.map { |part| part['text'] }.join }
                          .join

puts strategy_recommendations
```

## Part 2B Summary & Quick Reference

```ruby
# Part 2B Essential Commands
puts "\n📚 Part 2B Quick Reference"
puts "=" * 25

puts """
# Content Analysis
original_tweets = user.tweets.where(tweet_type: 'tweet')
replies = user.tweets.where(tweet_type: 'reply')

# Engagement Data
engagement_data = {
  avg_likes: user.tweets.average(:likes).to_f,
  avg_retweets: user.tweets.average(:retweets).to_f,
  consistency: (above_avg_count / total_count * 100).round(1)
}

# Fixed AI Generation with JSON Schema
result = GEMINI_CLIENT.stream_generate_content({
  system_instruction: system_instruction,
  contents: { role: 'user', parts: { text: prompt } },
  generation_config: {
    response_mime_type: 'application/json',
    response_schema: user_card_schema
  }
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

data = JSON.parse(json_string)

# Stats Formula Testing
engagement_attack = (avg_likes * 0.3 + avg_retweets * 0.7).clamp(1, 100)
influence_defense = (follower_ratio * 15 + consistency_bonus).clamp(1, 100)
"""

puts "\n✅ Part 2B Complete!"
puts "   • Tweet vs reply content patterns analyzed"
puts "   • Stats formulas tested with real engagement data"
puts "   • AI generation fixed with proper JSON schema"
puts "   • Character limit validation and retry logic implemented"
puts "   • Content strategy recommendations generated"
puts "\nReady for Part 2C: Controller Flow & Console Mastery"
```
