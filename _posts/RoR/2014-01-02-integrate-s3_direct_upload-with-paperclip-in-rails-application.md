---
layout: post
title: "Integrate S3_direct_upload with Paperclip in Rails Application"
description: "Direct uploading to Amazon S3 solution, build on S3_direct_upload and Paperclip "
category: "Ruby on Rails"
tags: [Ruby on Rails, Amazon AWS, Amazon S3]
---
{% include JB/setup %}


When I developed [Talent Pool Management Online Solution](http://www.talentlists.com) with Ruby on Rails, I picked [Paperclip](https://github.com/thoughtbot/paperclip) to manage my attachments on Amazon S3. It works very well except the uploading is slow. This is because my users had to upload the file to the web server, then the web server uploading the file to Amazon S3. This default behavior would also cause performance issues for Rails Deployment, [check this for details](https://devcenter.heroku.com/articles/s3).

[s3_direct_upload](https://github.com/waynehoover/s3_direct_upload) is a very powerful tool for uploading file directly to Amazon S3. When I was investigating the solution, I found [this blog](http://blog.littleblimp.com/post/53942611764/direct-uploads-to-s3-with-rails-paperclip-and).
This blog really helped me a lot. 

Since my Amazon S3 region is 'ap-northeast-1', I ran into some issues, I would record those adjustments in case someone else has the same troubles.

### Preparations
#### First add gems, then 'bundle install'
<pre><code class="ruby">
gem 'paperclip'
gem 'aws-sdk'
gem 's3_direct_upload'
</code></pre>

#### Setup CORS Configuration for your Amazon S3 bucket
{% highlight xml %}
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
       <CORSRule>
	      <AllowedOrigin>*</AllowedOrigin>
	      <AllowedMethod>POST</AllowedMethod>
	      <MaxAgeSeconds>3000</MaxAgeSeconds>
	      <AllowedHeader>*</AllowedHeader>
 	   </CORSRule>
</CORSConfiguration>
{% endhighlight %}

Check [the blog](http://blog.littleblimp.com/post/53942611764/direct-uploads-to-s3-with-rails-paperclip-and), If you need more instructions.


<!--more-->


#### Input your Amazon account credentials
I use [Figaro](https://github.com/laserlemon/figaro) to manage my enviroment-specific configurations.

My application.yml:
{% highlight ruby %}
     # ---  Amazon S3 and paperclip  ---
     access_key_id: xxxxxxxxxxxxx 
     secret_access_key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
	 s3_region: 'ap-northeast-1'

     development:
       bucket: sample.dev
     test:
       bucket: sample.test
     production:
       bucket: sample.prod
{% endhighlight %}

#### add a file: s3_direct_upload.rb to config/initializer
<pre><code>
 S3DirectUpload.config do |c|
   c.access_key_id = ENV['access_key_id']       
   c.secret_access_key = ENV['secret_access_key']
   c.bucket = ENV['bucket']
   c.region = "s3-"+ ENV['s3_region']   #Note: s3_direct_uplod need a "s3-" prefix
   end

   AWS.config({
   :access_key_id => ENV['access_key_id'],
   :secret_access_key => ENV['secret_access_key'],
   :region => ENV['s3_region']
   })
</code></pre>

<strong>Attention: s3_direct_uplod need a "s3-" prefix when setting region</strong>

### Controller,Views and Javascripts
   Follow the instruction on [s3_direct_upload](https://github.com/waynehoover/s3_direct_upload) and     [this blog](http://blog.littleblimp.com/post/53942611764/direct-uploads-to-s3-with-rails-paperclip-and)
   Here are my codes:
#### application.js
   
<pre><code>
//= require s3_direct_upload
</code></pre>

####  application.css
{% highlight ruby %}
 *= require_self
 *= require s3_direct_upload_progress_bars
 *= require custom
{% endhighlight %}

#### View:
{% highlight ruby %}
{% raw %}
<div class="row">
    <div class="span6 file-input">
	<%= s3_uploader_form callback_url: talent_attachments_url,
 	                     callback_param: "temp_upload_path", 
	                     key: "temp_uploads/{timestamp}_{unique_id}/${filename}",
	                     key_starts_with: "temp_uploads/",
	                     expiration: 24.hours.from_now.utc.iso8601,
                             max_file_size: 10.megabytes,
	                     acl: "private", 
	                     id: "s3-uploader" do %>

	   <%= file_field_tag :file, multiple: true, size: 40, style:"width: 450px;" %>
        <% end %>

        <script id="template-upload" type="text/x-tmpl">
            <div id="file-{%=o.unique_id %}" class="upload">
	     {%=o.name%}
	    <div class="progress"><div class="bar" style="width: 0%"></div></div>
            </div>
        </script>
    </div>
    <div class="span6">
	<table class="table table-condensed" id="attachment-list">
	    <% @talent.attachments.each do |att| %> 
		<%= render :partial => "talent_attachments/line", :locals => {:attachment => att} %>
	    <% end %> 
	</table>
    </div>
</div>
{% endraw %}
{% endhighlight %}

####   partial:
{% highlight ruby %}
<% if attachment.attachment_updated_at %>
    <tr>
  	<td><%= link_to truncate(attachment.attachment_file_name), download_talent_attachment_path(attachment) %></td>
	<td><%= number_to_human_size(attachment.attachment_file_size) %></td>
	<td><%= attachment.attachment_updated_at.to_s(:dhm) %></td>
	<td><%= link_to 'Delete', attachment, :method => :delete, :confirm => "You sure?"%></td>
    </tr>
<% end %>
{% endhighlight %}


####   controller
{% highlight ruby %}
  def create
    talent = current_company.talents.find(params[:talent_id])
    temp_upload_path = CGI.unescape(params[:temp_upload_path]) rescue nil
    @attachment = talent.attachments.build(upload_path: temp_upload_path)
    @attachment.update_attachment_attributes

    if @attachment.save 
      if @attachment.finalize_s3_file
        flash[:success]= 'Attachment uploaded successfully'
      else
        remove_temp_file(@attachment)
        flash[:error] = "Uploading failed, please try again."
      end
    else
      flash[:error] = @attachment.errors.full_messages[0] if @attachment.errors.count > 0
    end
  end

  def destroy
    @attachment = TalentAttachment.find(params[:id])
    talent = @attachment.talent
    @attachment.destroy
    
    redirect_to talent, :flash =>{:success => 'Deleted successfully.'}
  end

  def download
    @attachment = TalentAttachment.find(params[:id])
    redirect_to @attachment.attachment.expiring_url(30)
  end
{% endhighlight %}

  


####   Model:
{% highlight ruby %}
class TalentAttachment < ActiveRecord::Base
  attr_accessible :upload_path

  has_attached_file :attachment,
                    :path => "/:class/:id_partition/:filename",
                    :s3_permissions => :private
  belongs_to :talent

  UPLOAD_PATH_FORMAT = %r{\Ahttps:\/\/s.+\.amazonaws\.com\/.+\/(?<path>temp_uploads\/.+\/(?<filename>.+))\z}.freeze
  validates :upload_path, presence: true, format: { with: UPLOAD_PATH_FORMAT }
  validate :validate_max_count
  validates_attachment :attachment, :presence => true, :size => { :less_than => 10.megabytes }

  def finalize_s3_file
    begin
      upload_url_data = UPLOAD_PATH_FORMAT.match(self.upload_path)
      file_path_no_slash = self.attachment.path[1..-1]

      s3 = AWS::S3.new
      s3.buckets[ENV['bucket']].objects[file_path_no_slash].copy_from(upload_url_data[:path])
      s3.buckets[ENV['bucket']].objects[upload_url_data[:path]].delete
      return true
    rescue AWS::S3::Errors::Base => e
      return false
    end
  end

  def remove_temp_file
    s3 = AWS::S3.new
    s3.buckets[ENV['bucket']].objects[attachment.upload_path].delete
    self.destroy
  end

  def update_attachment_attributes
    tries ||= 3
    upload_url_data = UPLOAD_PATH_FORMAT.match(self.upload_path)

    s3 = AWS::S3.new
    file_head = s3.buckets[ENV['bucket']].objects[upload_url_data[:path]].head
   
    self.attachment_file_name = upload_url_data[:filename]
    self.attachment_file_size = file_head.content_length
    self.attachment_content_type = file_head.content_type
    self.attachment_updated_at = file_head.last_modified
  rescue AWS::S3::Errors::Base => e
    tries -= 1
    if tries > 0
      sleep(3)
      retry
    else
      false
    end
  end


  protected
  MAX_ATTACHMENT_COUNT = 5

  def validate_max_count
    if self.talent.attachments(:reload).count >= MAX_ATTACHMENT_COUNT
      errors.add(:base, "Too many attachments - maximum is #{MAX_ATTACHMENT_COUNT}") 
    end
  end
 end
{% endhighlight %}

### Update view
   In order to update the views when the uploading was completed, a "create.js.erb" is required.
{% highlight ruby %}
   <% if @attachment.new_record? %>
  alert("Failed to upload files: <%= j @attachment.errors.full_messages.join(', ').html_safe %>");
<% else %>
  $("#attachment-list").append("<%= j render(:partial =>'line', :locals => {attachment: @attachment}) %>");
<% end %>
{% endhighlight %}

