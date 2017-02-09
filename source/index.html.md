---
title: DataGene Smartphone

language_tabs:
  - ruby
  - javascript

toc_footers:
  - <a href='http://datagene.com.au'>Farm Management app <br> developed by DataGene</a>

search: true
---

# Introduction

>DataGene uses the Rhodes Cross platform mobile framework to build the cross platform
application for iOS / Android.


Welcome to the DataGene Smartphone documentation.

This will guide you through the endpoints for requesting / posting data in the controllers locally to display on screen using whichever HTML/Javascript framework is in place (currently Framework7) and sync processes to the backend server.

# Sign in - Existing User
## Internal

```ruby
 # $UserName and $Pass variables are set based on query string params sent from the HTML form,
 # then IP addresses used for sync are fetched with asyncget_ip().

  def do_normal_login
    $fromlogin = true
    $UserName = @params['username']
    $Pass = @params['password']
    asyncget_ip()
  end
```

```javascript

// Verify fields before submitting form to do_normal_login.

function LoginValidate() {
	if (document.getElementById('username').value == "")
	{
		myApp.alert('You must enter your Email Address', 'Missing Information!');	
		return false;
	}
	var mailformat = /^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/;  
	if(document.getElementById('username').value.match(mailformat))  
	{    
		//continue.
	}  
	else  
	{  
		myApp.alert("You have entered an invalid email address!",'Invalid Email Address!');  
		document.getElementById('username').focus();  
		return false;  
	}  

	if (document.getElementById('password').value == "")
	{
		myApp.alert('You must enter a password', 'Missing Information!');
		return false;
	}

	// if all is ok then submit the form.
	document.getElementById('password').value = md5(document.getElementById('password').value.toUpperCase());
	document.getElementById("LoginForm").submit();
	return true;			
}
```

This endpoint accepts POST requests in the Ruby controller from an HTML form.
It then fetches IP addresses of DPC's and forwards to the external section.

### HTTP Request
`POST /app/Settings/do_normal_login`

Parameter | Description
--------- | -----------
username | Username / E-Mail address used in DataGene Web / Mobile registration.
password | Password used in DataGene Web / Mobile registration.

## External 

```ruby
# Use Rhodes AsyncHTTP.get to request data to server.

  def login_toserver
    Rho::AsyncHttp.get(
      :url => "#{$serveraddress}/pktuser-new.json?username=#{$UserName}&password=#{$Pass}",
      :authentication => {
           :type => :basic, 
           :username =>"vn5BdmRurZP8uHfk", 
           :password => "JkBmVSTJGemPu9P9"
         },
      :callback => (url_for :action => :login_callback)
    )
    render :action => :wait_login  
  end
```

  > /pktuser-new endpoint returns JSON structured like this:

```json
[  
   {  
      "pktuser":{  
         "Users_PocketKey":"guy0yxeh00gj",
         "Users_DBID":"999",
         "Users_Password":"549030d43960c0b5b505bdc4da93f6e5",
         "Herd":"703293"
      }
   }
]
```

This endpoint accepts requests from the app and checks for users.

### HTTP Request
`GET http://202.164.192.189:3001/pktnew-user.json`

### Query Parameters

Parameter | Description
--------- | -----------
username | Username / E-Mail address used in DataGene Web / Mobile registration.
password | Password used in DataGene Web / Mobile registration.

<aside class="warning">
Remember — 403 error means username or password is incorrect!
</aside>

# Register new user

## Internal

```ruby
# Store Pktoption params in local database, then fetch IP addresses.

  def asyncregisterget_ip
    if System.has_network()
      @pktoptionupdate = Pktoption.find(:first)
      if @pktoptionupdate
        Pktoption.delete_all
      end
      @pktupdate = Pktupdate.find(:all)
      if @pktupdate
        Pktupdate.delete_all
      end
      @pktoption = Pktoption.create(@params['pktoption']) 
      @pktupdate = Pktupdate.create(@params['pktupdate'])      
      Rho::AsyncHttp.get(
      :url => "#{$ipaddress_json}",
      :callback => (url_for :action => :registerip_callback),
      :callback_param => "" )
      render :action => :wait
    end
  end
```

```ruby
  # Store IP addresses.

  def registerip_callback
    if @params["status"] != "ok"
        @@error_params = @params
        WebView.navigate( url_for(:action => :show_error) )        
    else
      @data = Rho::JSON.parse(@params["body"])
      puts "DOWNLOADED #{@data}"
      Pktip.delete_all
      db = ::Rho::RHO.get_src_db('Pktip')
      db.start_transaction
      begin
        @data.each do |ip|
          myip = Pktip.new
          myip.syncipaddress = ip["pktip"]["syncipaddress"]      
          myip.postipaddress = ip["pktip"]["postipaddress"]      
          myip.centre = ip["pktip"]["Centre"]  
          myip.name = ip["pktip"]["Name"]    
          myip.save
        end
       db.commit
      end
    end
      WebView.navigate( url_for :action => :check_user )  
  end
```

```javascript
// Validate fields and submit to asyncregisterget_ip.

function LoginValidateRegister() 
{
	if (document.getElementById('reg-password').value != "")
    {
		document.getElementById('reg-password').value = document.getElementById('reg-password').value.toUpperCase();
        document.getElementById('md5password').value = md5(document.getElementById('reg-password').value);
    }
	document.getElementById('tech').value = document.getElementById('email').value.substr(0, document.getElementById('email').value.indexOf('@'));			
	document.getElementById('pktoptions_username').value = document.getElementById('email').value;

	document.getElementById('pktupdates_sync').value = document.getElementById('pktupdates_sync').value.toUpperCase();

	// if all is ok then submit the form.
	document.getElementById("RegisterForm").submit();
	return true;
}
```

This endpoint accepts requests from registration forms, fetches IP addresses of DPC's and passes through to external server for processing.

### HTTP Request
`GET /app/Settings/asynchttp_register`

### Query Parameters

Parameter | Description
--------- | -----------
pktoptions_username | E-Mail address for registration
md5password | md5 hashed password for registration
pktupdates_sync | Herd Mask
tech | Farm tech set to first part of email address

## External

```ruby  
# Check User with usercheck to see if username exists and determine DPC for herd.

  def check_user
    if System.has_network() 
      $httpresult = "Checking username"
			data = Pktoption.find(:first) 
			$UserName = data.Pktoptions_Email
      $Herds = data.Pktoptions_HerdIDs
	
      Rho::AsyncHttp.get(
        :url => "#{$serveraddress}/usercheck.json?login=#{$UserName}&herds=#{$Herds}",
        :callback => (url_for :action => :checkuser_callback),
        :authentication => {
             :type => :basic, 
             :username =>"vn5BdmRurZP8uHfk", 
             :password => "JkBmVSTJGemPu9P9"
           },
        :callback_param => "" )
        render :action => :wait
    end
  end
```

```ruby
# If Users_Email in returned JSON then user exists.
# If DPC in returned JSON set DPC based on what is returned.

  def checkuser_callback
    if @params['body'].include? 'Users_Email'
      $HerdID = 'None'
      @pktoption = Pktoption.find(:first)
      @pktoption.update_attributes({"Pktoptions_HerdIDs" => "None"})
      alertBox("A User with that email address: #{$UserName} already exists.
       Please user another.",'User Exists')
      WebView.navigate('/app/Settings/new') 
    else
      $dpc = "#{@params['body']}"
      if $DemoVersion == true
        $dpc = 999
      end
      @pktoption = Pktoption.find(:first)        
      @pktoption.update_attributes({"Pktoptions_HTCentre" => "#{$dpc}"})
      WebView.navigate( url_for :action => :do_login ) 
    end
  end
```

```ruby
# JSON generate PktOptions table and send to server.

  def do_login
    if System.has_network() 
			data = Pktoption.find(:first) 
			gen_content = "\"pktoption\":" + ::JSON.generate(data)
			$HerdID = data.Pktoptions_HerdIDs
			$Techs = data.Pktoptions_TechNames
			$UserName = data.Pktoptions_UserName
			@HTCentre = data.Pktoptions_HTCentre
			@clientid = data.Pktoptions_id
			@pass = data.Pktoptions_Password
      $httpresult = 'Sending request'
      @pktip = Pktip.find(:first, :conditions => ['centre like ?',
       "#{@HTCentre}"], :select => ['centre','postipaddress'])
        if @pktip
          @serveraddress = @pktip.postipaddress
        else
          @serveraddress = "202.164.192.189:9890"
        end
          Rho::AsyncHttp.post(
           :url => "#{@serveraddress}",
				   :headers => {'Content-Type'=>'application/json'},
				   :authentication => {
					  :type => :basic, 
					  :username => "register", 
					  :password => "register"
					 },
				   :body => gen_content, 
				   :callback => (url_for :action => :httplogin_callback), 
				   :callback_param => "" )
     
        render :action => :wait
    else
      okBox("You are not connected to the internet! 
      Cannot Sync Data!",'/app/Start/index','No Internet') 
    end
  end
```

> PocketService endpoint returns a body text of 'All OK.' if successful.



This endpoint checks the User is available on the server and returns the DPC number to be used for sensing registration.

### CHECK USER HTTP Request
`GET http://202.164.192.189:3001/usercheck.json`

### Query Parameters

Parameter | Description
--------- | -----------
login | E-Mail address for registration
herds | Herd list user is registering



This 'PocketService' endpoint sends the registration request to the server.

### SEND Registration request
`POST http://202.164.192.189:9890`

### Query Parameters

Parameter | Description
--------- | -----------
Pktoptions_UserName | E-Mail address for registration
Pktoptions_Password | Password for registration
Pktoptions_HerdIDs | Herd list user is registering
Pktoptions_TechNames | Name of tech (email address before '@')
Pktoptions_HTCentre | DPC number
Pktoptions_id | client id generated for device

# Cow Lists

## General Listings
```javascript

```

```ruby

```

This generalizes the list endpoints and the returned JSON data structure.

### CowList HTTP Request
`GET /app/Pktfemale/find_cows`

## Search functions
```javascript
```

```ruby
```

This generalizes the search endpoints when typing an animal ID and pressing done.

### Search HTTP Request
`GET /app/Pktfemale/show`

Parameter | Description
--------- | -----------
search | Herd Recording number of animal