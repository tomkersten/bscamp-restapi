!SLIDE
# Thoughts on "RESTful" APIs #
### @tomkersten

!SLIDE
# Thoughts on RESTful(-ish) APIs #
### @tomkersten

!SLIDE
# REST !=  PUT/POST/DELETE
### most apis lack "hypermedia controls" (forms, links)

!SLIDE
# read this:
## "Building Hypermedia APIs with HTML5 and Node"

!SLIDE
# /disclaimer
### moving on...

!SLIDE
# background:
## we needed to share some files

!SLIDE
# background:
## we needed to share some files
### dropbox, et al were not acceptable

!SLIDE
# enter: RAM

!SLIDE
# enter: RAM
### that's what we called it.

!SLIDE bullets incremental
# topics

* api versioning
* named endpoints
* misc (tools, tips, etc)

!SLIDE

# api versioning

!SLIDE
# popular approaches
### none (fuggit...we're agile), url-based, mime-type based

!SLIDE
# vendor-specific mime types
## using the ('accept' header)

!SLIDE
# vendor-specific mime types
### all the cool kids are doing it...
### ([and it's kind of in the mime spec](http://tools.ietf.org/html/rfc2048#section-2.1.2))

!SLIDE
# vendor-specific mime types
### Accept: application/vnd.company-name.app-name+format

!SLIDE
# vendor-specific mime types
### Accept: application/vnd.resound.ram+json

!SLIDE
# where's the versioning?

!SLIDE
# where's the versioning?
### Accept: application/vnd.resound.ram+json;version=2

!SLIDE
# one approach

!SLIDE
## (quick sidenote)
## Mime::Type.register "application/pdf", :princely
### (maps "/resource-id.pdf" &#8594; "show.princely")


!SLIDE
# One approach
### settings.yml &#8594; mime_type_registrar.rb &#8594; mime_types.rb

!SLIDE code

    @@@ ruby
    # config/settings.yml
    supported_api_versions: [3]
    deprecated_api_versions: [2]
    unsupported_api_versions: [1]


!SLIDE code smaller

    @@@ Ruby
    # lib/ram_api.rb
    # light wrapper for API-values in settings.yml
    module RamAPI
      def current_version
        Settings.supported_api_versions.max
      end

      def supported_versions
        Settings.supported_api_versions
      end
      ...
    end

!SLIDE code smaller

    @@@ Ruby
    # lib/ram_api.rb
    # light wrapper for API-values in settings.yml
    module RamAPI
      ...
      def deprecated_versions
        Settings.deprecated_api_versions
      end

      def deprecated_version?(version)
        deprecated_versions.include?(version.to_i)
      end
    end

!SLIDE code smaller

    @@@ Ruby
    # lib/mime_type_registrar.rb
    # wraps up mime type registration and tracks deprecations
    module MimeTypeRegistrar
      extend self
      BASE_MIME_TYPE = "application/vnd.resound.ram+json"

      def versioned_mime(version_number)
        "#{BASE_MIME_TYPE};version=#{version_number}"
      end
      ...

!SLIDE code smaller

    @@@ Ruby
    # lib/mime_type_registrar.rb
    # wraps up mime type registration and tracks deprecations
    module MimeTypeRegistrar
      ...
      # wrapped in convenience methods
      def register(type, file_ext)
        Mime::Type.register type, file_ext.to_sym

        ActionView::Template.register_template_handler file_ext,
            ActionView::Template::Handlers::JsonifyBuilder

        self.supported_content_types << type
      end
      private :register
    end

!SLIDE code smaller

    @@@ Ruby
    # config/initializers/mime_types.rb
    # Set up mime types for app
    include MimeTypeRegistrar

    register_default_api_version(RamAPI.current_version)

    RamAPI.supported_versions.each do |version|
      register_api_version(version)
    end

    RamAPI.deprecated_versions.each do |version|
      register_deprecated_api_version(version)
    end

!SLIDE

# what does that give us?


!SLIDE

## Accept: "vnd/resound.ram+json;version=2"
### renders: "show.json_v2"


!SLIDE

## Accept: "vnd/resound.ram+json"
### renders: "show.json_[current_version]"


!SLIDE

# Clients lock in on a version
## ...and don't break on your updates*


!SLIDE

# Clients lock in on a version
## ...and don't break on your updates&#8727;
### &#8727; unless your domain model changes significantly



!SLIDE

## bonus: layouts for free
### ...nested in versioned layout files (layouts/application.json_v2)


!SLIDE code smaller

    @@@ Ruby
    # app/views/layouts/application.json_v1
    json.content do
      json.ingest! yield
    end

    # app/views/layouts/application.json_v2
    json.content do
      json.ingest! yield
      json.tx_meta @tx_meta
      json.developer_notes @developer_notes
    end


!SLIDE

## bonus: "free" deprecation info


!SLIDE code smaller

    @@@ Ruby
    class ApplicationController < ActionController::Base
      before_filter :check_for_deprecated_api_version

      ...
      def check_for_deprecated_api_version
        if RamAPI.deprecated_version?(req_vrsion)
          developer_notes << "v#{req_vrsion} of API is deprectd."
          developer_notes << "Please upgrade v#{cur_vrsn}."
        end
      end
      ...
    end


!SLIDE

## bonus: easy to enforce supported versions


!SLIDE code smaller

    @@@ Ruby
    class ApplicationController < ActionController::Base
      ...
      def enforce_supported_formats
        render_not_acceptable && return unless supported_format_requested?
      end

      def supported_format_requested?
        MimeTypeRegistrar.supported_content_type?(request.format)
      end
      ...
    end

!SLIDE

# /api versioning
### questions/thoughts?


!SLIDE

# Named endpoints


!SLIDE

# Named endpoints
## wtf are they?


!SLIDE

# hateoas
### hypermedia as the engine of application state

!SLIDE

# what?


!SLIDE

# hypermedia =~ "links"
### stop hardcoding URLs in your clients

!SLIDE

# hypermedia =~ "links"
### stop hardcoding URLs in your clients
### there's more to it than that, but...


!SLIDE

# an example...


!SLIDE code smaller

    @@@ JavaScript
    // app/views/landing/index.json_v3 (rendered)
    {
      "content": {
        "links": {
          "create_collection": {
            "url": "/collections"
          },
          "list_collections": {
            "url": "/collections"
          }
        }
      }
    }

!SLIDE

# want to see all collections?
### hit the URL at `content.links.list_collections.url`

!SLIDE

# it's that simple...
### use it* for all your links

!SLIDE

# why?


!SLIDE

# freedom to change urls on server


!SLIDE

## if you're really good (organized)...
### you can optimize slow areas


!SLIDE

## if you're really good (organized)...
### you can optimize slow areas
### ...without breaking any clients


!SLIDE



!SLIDE

## note: if you put "/collections" in your library...
### your fingers get broken.


