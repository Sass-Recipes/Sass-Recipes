# Sass, Sinatra, and Heroku

Or, how I built SassMeister.com

======

## `rake deploy:awesome`

This is fine during development:

    configure do
      Compass.add_project_configuration(File.join(Sinatra::Application.root, 'config.rb'))
    end
  
    get '/stylesheets/:name.css' do
      content_type 'text/css', :charset => 'utf-8'
      scss(:"../sass/#{params[:name]}", Compass.sass_engine_options )
    end

But don't let me catch you using it in production!

This route pass all requests for `example.com/stylesheets/*.css` through the `scss` [template engine](http://www.sinatrarb.com/intro.html#SCSS%20Templates). Every. Request. Which is very bad for performance in production.

What to do? 

Joe handles this by compiling the Sass on his dev box. But then realizes that he can't push the compiled CSS to Heroku because he followed our [earlier advice](##) about omitting compiled CSS from version control. (files not pushed with `git push`, etc...) Joe is desparate to deploy his shiny new app, so he removes "/stylesheets" from his .gitignore, commits the compiled CSS and pushes to production. 

Jill, however, is lazy, as a good programmer should be; she'd rather have the server handle compilation for her. And, like a good Rubyist, she knows Rake is her friend. She discovers that Heroku uses the Rails asset pipeline .... `rake asset:precompile` is called during the build process .... can be used outside of Rails. 


https://devcenter.heroku.com/articles/rails3x-asset-pipeline-cedar


Jill adds the following to app/Rakefile:

    # Heroku will run this task as part of the deployment process.
    desc "Compile the app's Sass"
    task "assets:precompile" do
      system("compass compile")
    end

commits, runs `git push heroku`, and gets a promotion!

======


Explain: 

    
    post '/compile' do
    
    # Adds the Compass load paths to the Sass.load_paths array
    # making all registered Compass extensions available for @import'ing
      Compass.sass_engine_options[:load_paths].each do |path|
        Sass.load_paths << path
      end
    
    
    # Examine the POST'ed extension value, and create the @import string accordingly
      if params[:extension] == 'bourbon'
        extension = "@import \"bourbon/bourbon\""
      elsif params[:extension] == 'compass'
        extension = "@import \"compass\""
      elsif params[:extension] == 'stipe'
        extension = "@import \"./sass/stipe\""
      else
        extension = ''
      end
    
    
    # Concatenate the extension @import string with the POST'ed Sass value
    # adding a semi-colon after the @import statement if necessary
      if ! extension.empty? and params[:syntax] == 'scss'
        sass = "#{extension};\n\n#{params[:sass]}"  # semi-colon after @import for SCSS syntax
      else
        sass = "#{extension}\n\n#{params[:sass]}"   # NO semi-colon for Sass syntax
      end
    
    
    # Use the Ruby [`send` method](http://ruby-doc.org/core-1.9.3/Object.html#method-i-send) switch between the two Sass syntaxes as necessary
    # Roughly equivilent to:
    #    
    #    if params[:syntax] == 'sass'
    #      sass(params[:sass], {:style => :"#{params[:output]}"})
    #    else
    #      scss(params[:sass], {:style => :"#{params[:output]}"})
    #
      begin
        send("#{params[:syntax]}".to_sym, sass.chomp, {:style => :"#{params[:output]}"})
        
     
    # In the event of a Sass syntax error, catch it and display it so the user knows what went wrong.
      rescue Sass::SyntaxError => error
        status 200  # Force an HTTP status of 200 so that jQuery will recieve the response as "successful". Without this, Sinatra returns 500 and jQuery won't insert the return into the DOM.
        error.to_s
      end
    end













