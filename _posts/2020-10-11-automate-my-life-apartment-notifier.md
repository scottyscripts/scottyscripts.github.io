---
layout: post
title:  "Automate My Life - Using AWS to Create Custom Apartment Notifications"
date: 2020-10-11
---

## I Hate Moving

I recently decided to move... again. I love to travel, explore, and experience the many diverse parts of our country, but I absolutely **HATE** looking for apartments.

Apartment hunting is probably the easiest it's ever been in 2020, but it can still be a pain. During this search, I noticed that many places' prices weren't accurate if I wasn't checking on the actual property management / apartment building's site. I soon became annoyed checking multiple property management sites every day to track availability and price changes. Many of the sites I was looking at didn't have an option to sign up for email notifications, so... I decided to create my own!

## No Notifications? No Problem

After breaking down the problem, I knew that I would need to:
- scrape apartment data from a website
- store that data
- determine any price / availibility changes with any previously stored data
- send an email notifying me about the changes and what they were
- automate the entire process to run every morning

At the time, I was working on a Ruby on Rails project, so I made the decision to do this with Ruby.

After doing some more research, I decided to use the following tools:
- [Nokogiri](https://nokogiri.org/) to parse the HTML of property management site with the apartment listings
- [Amazon S3](https://aws.amazon.com/S3/) via the [AWS SDK for Ruby](https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/setup-config.html) to store the apartment data
- [SendGrid](https://sendgrid.com/) via the [sendgrid-ruby gem](https://github.com/sendgrid/sendgrid-ruby) to send me email updates
- [Amazon Lambda](https://aws.amazon.com/lambda/) to run my code
- [Amazon EventBridge](https://aws.amazon.com/eventbridge/) (formerly CloudWatch Events) to trigger my code to run every morning

I went back and forth of where / how to run my code. When I realized I could run this for practically free on AWS, it was a no brainer. 

Before I could get this running on AWS, I wanted to get it working locally. I ended up starting with some pseudocode that looked like this.

```ruby
# 1) parse HTML and create JSON apartment data

# 2) get apartment data from S3 from previous time script ran (if it's run before)

# 3) compare old data with new data, get changes
#    - new apartments
#    - price changes
#    - availability changes
#    - removed apartments

# 4) send email to myself with changes in apartment data

# 5) save newest apartment data to S3
```

## I came, I 鋸 - parsing HTML with Nokogiri

I had never actually directly used the `nokogiri` gem in Ruby but had always seen it mentioned (mostly in error messages). Nokogiri is a pretty large library for parsing, querying, and editing documents, but I was just interested in using some of its super basic functionality. Below is the code I used to parse the HTML of the site with the apartment listings.

```ruby
url = '<URL with apartment listings>'
# parse HTML and assign resulting 'Nokogiri document' to var
doc = Nokogiri::HTML(open(url))

# using css method to query for all elements with specified CSS selector
# result is 'Nokogiri NodeSet'; acts like Array that contains matching nodes
apartment_containers_html = doc.css('li.apartment-card')

# create list of apartment info from each node (grabbing title, unit, details, price, availability values from HTML)
apartments_info = apartment_containers_html.map do |apartment_container_html|
  info = apartment_container_html.css('div.content')

  title = info.css('div.title').text
  unit_number = title.split(' ').last
  details = info.css('div.details').text
  price = info.css('div.price').text
  availability = info.css('div.availability').text

  { 
    unit_number: unit_number,
    price: price,
    availability: availability
  }
end
```

## Getting Previous Data - Setting Up S3

Now that I had the most recent apartment data scraped from the site, I wanted to compare that to the data from the previous time my code ran. I noticed that prices were changing and new apartments were being added daily, and I didn't want to miss anything.

I created a new S3 bucket named `apartment-scraper` and then created `apartments.json` inside. This file contained an empty JSON Array `[]` but would eventually contain the actual apartment data.

The next step was retrieving this `apartments.json` file and comparing the data within to the most recently obtained apartment data. 

I set the following environment variables locally ([AWS docs - Setting Credentials Using Environment Variables](https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/setup-config.html#aws-ruby-sdk-credentials-environment))
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

> Note: These will automatically be set when deployed on AWS Lambda ([more info](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html#configuration-envvars-runtime))

After configuring my credentials, I ended up with the following.

{% highlight ruby %}
# instantiate new S3 client
s3 = Aws::S3::Client.new(region: 'us-east-2')

# get apartment data from S3
res = s3.get_object({
  bucket: 'apartment-scraper',
  key: 'apartments.json'
})

# parse data from apartments.json
old_apartments_info = JSON.parse(
  res.body.read,
  {symbolize_names: true}
)
{% endhighlight %}

Now I had the new and old apartment data. The next step was to compare them.

## Ch-ch-ch-ch-changes

This next part just required some Ruby code to compare my two lists of apartment data.

<details>
<summary>Comparing Apartment Data Code</summary>
<br>

{% highlight ruby %}
new_apartments = []
price_changes = []
availability_changes = []
no_changes = []
removed_apartments = []

apartments_info.each do |apartment_info|
  unit_number = apartment_info[:unit_number]
  price = apartment_info[:price]
  availability = apartment_info[:availability]

  done = false

  old_apartments_info.each_with_index do |old_apartment_info, i|
    break if done
    
    has_same_unit_number = old_apartment_info.has_value?(unit_number)
    has_same_price = old_apartment_info.has_value?(price)
    has_same_availability = old_apartment_info.has_value?(availability)

    is_last = i == (old_apartments_info.size - 1)

    case
      # nothing changed
      when has_same_unit_number && has_same_price && has_same_availability
        no_changes << apartment_info
        done = true
      # price changed
      when has_same_unit_number && has_same_availability
        price_changes << apartment_info
        done = true
      # availability changed
      when has_same_unit_number && has_same_price
        availability_changes << apartment_info
        done = true
      # availability and price changed... so just notify on price diff
      when has_same_unit_number
        price_changes << apartment_info
        done = true
      when is_last
        new_apartments << apartment_info
      else
        nil
    end
  end
end

removed_apartments = old_apartments_info - apartments_info
{% endhighlight %}
</details>

## Sending An Email

I followed the [setup instructions](https://github.com/sendgrid/sendgrid-ruby#setup-environment-variables) for the `sendgrid-ruby` gem (setting environment variable of the API key and adding `include SendGrid` to my code). You can sign up for SendGrid for free and easily get an API key!

I then created an ERB template for my email,

<details>
<summary>ERB Email Template</summary>
<br>

{% highlight erb %}
<div>
  <p>Apartment Updates for Today</p>
</div>

<div class="container">
  <div>
    <p>These are (apartment building name) apartment listings for today.</p>
  </div>
  <div>
    <% if new_apartments.length > 0 %>
      <h2>New Apartments</h2>
      <% new_apartments.each do |apartment| %>
        <h3><%= apartment[:unit_number] %></h3>
        <ul>
          <li><%= apartment[:price] %></li>
          <li><%= apartment[:availability] %></li>
        </ul>
      <% end %>
    <% else %>
      <h2>No New Apartments</h2>
    <% end %>

    <% if price_changes.length > 0 %>
      <h2>Price Changes</h2>
      <% price_changes.each do |apartment| %>
        <h3><%= apartment[:unit_number] %></h3>
        <ul>
          <li><%= apartment[:price] %></li>
          <li><%= apartment[:availability] %></li>
        </ul>
      <% end %>
    <% end %>

    <% if availability_changes.length > 0 %>
      <h2>Availability Changes</h2>
      <% availability_changes.each do |apartment| %>
        <h3><%= apartment[:unit_number] %></h3>
        <ul>
          <li><%= apartment[:price] %></li>
          <li><%= apartment[:availability] %></li>
        </ul>
      <% end %>
    <% end %>

    <% if removed_apartments.length > 0 %>
      <h2>Removed Apartments</h2>
      <% removed_apartments.each do |apartment| %>
        <h3><%= apartment[:unit_number] %></h3>
        <ul>
          <li><%= apartment[:price] %></li>
          <li><%= apartment[:availability] %></li>
        </ul>
      <% end %>
    <% end %>

    <% if no_changes.length > 0 %>
      <h2>No Updates On These</h2>
      <% no_changes.each do |apartment| %>
        <h3><%= apartment[:unit_number] %></h3>
        <ul>
          <li><%= apartment[:price] %></li>
          <li><%= apartment[:availability] %></li>
        </ul>
      <% end %>
    <% end %>
  </div>
</div>
{% endhighlight %}

</details>

Then, I wrote the Ruby code to send the email using my template

{% highlight ruby %}
email_template = File.open('./templates/email.erb').read

# pass data into ERB template
html_email = ERB.new(email_template).result_with_hash({
  new_apartments: new_apartments,
  price_changes: price_changes,
  availability_changes: availability_changes,
  no_changes: no_changes,
  removed_apartments: removed_apartments,
})

subject = 'Apartment Updates'
# store 'from' and 'to' emails in ENV variable so it can easily be changed
from = SendGrid::Email.new(email: ENV['EMAIL_FROM'])
to = SendGrid::Email.new(email: ENV['EMAIL_TO'])
content = SendGrid::Content.new(
  type: 'text/html',
  value: html_email
)
mail = SendGrid::Mail.new(from, subject, to, content)

# instantiate SendGrid client
sg = SendGrid::API.new(api_key: ENV['SENDGRID_API_KEY'])
# send email and handle repsonse
response = sg.client.mail._('send').post(request_body: mail.to_json)
if /^2/.match(response.status_code)
  puts "\nSuccessfully sent email\n"
else 
  puts"\nFailed to send email\n"
end
{% endhighlight %}

This was also where I decided to upload the 'new' apartment data to S3. This operation overwrites the data that was previously in the file hosted on S3.

{% highlight ruby %}
s3.put_object({
  bucket: 'apartment-scraper',
  key: 'apartments.json',
  content_type: 'application/json',
  body: apartments_info.to_json
})
{% endhighlight %}

## Deploying to AWS Lambda

AWS Lambda allows you to create a function and write code in their GUI editor, but you are also able to upload a [deployment package](https://docs.aws.amazon.com/lambda/latest/dg/ruby-package.html) (.zip file containing code and dependencies)

I did a little refactoring of my script.

{% highlight ruby %}
require 'nokogiri'
require 'open-uri'
require 'json'
require 'sendgrid-ruby'
require 'erb'
require 'aws-sdk-s3'

# will eventually tell Lambda to invoke this method
def lambda_handler(event:, context:)
  ApartmentNotifier.run()
end

class ApartmentNotifier
  include SendGrid

  def self.run() 
    # all my code

{% endhighlight %}

I created a zip archive of my function and deps by following the guide for [deploying a Ruby function with dependencies](https://docs.aws.amazon.com/lambda/latest/dg/ruby-package.html#ruby-package-dependencies).

Then, I [created a new Lambda function on AWS](https://docs.aws.amazon.com/lambda/latest/dg/getting-started-create-function.html) (I chose to use the `Ruby 2.7` runtime since that is what I was running locally when I wrote this code) and uploaded my `.zip` file under the `function code` section.

I had to set all of my environment variables and edit the 'basic settings' to specify my handler method as `main.lambda_handler`. This was my value because:
1. the file my code was in was `main.rb`
2. I defined a method named `lambda_handler` (this is the actual method I wanted to run)

## Automate the Process

After making sure that my lambda function was working by [running it manually](https://docs.aws.amazon.com/lambda/latest/dg/getting-started-create-function.html#get-started-invoke-manually), I went over to AWS `EventBridge`.

> I mentioned this before, but EventBridge was formerly CloudWatch Events.

I [created an EventBridge rule](https://docs.aws.amazon.com/eventbridge/latest/userguide/create-eventbridge-rule.html), using the [cron expression pattern option](https://docs.aws.amazon.com/eventbridge/latest/userguide/scheduled-events.html#cron-expressions). This told my event to run every morning.

I made my newly created Lambda function the target of my event.

## Be Lazy, Automate Things

The next morning, I received an email of all the updated apartment listings. I knew about any price and availability changes every day, which allowed me to act fast.

I love automation and coding, and this is an example of how I incorporate that into my daily life. This took a very short amount of time and made my apartment hunt so much easier.

I hope this inspired you to consider how you can start automating your life. Both at work and in my daily life, I've found that investing a short amount of time in automation can make a world of difference.

If you would like a reference to get started, you can find [all of my code here](https://github.com/scottyscripts/Apartment-Notifier)!