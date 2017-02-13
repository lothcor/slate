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

This endpoint accepts POST requests in the Ruby controller from an HTML form.
It then fetches IP addresses of DPC's and forwards to the external section.

### HTTP Request

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

`POST /app/Settings/do_normal_login`

#### Query Parameters

Parameter | Description
--------- | -----------
username | Username / E-Mail address used in DataGene Web / Mobile registration.
password | Password used in DataGene Web / Mobile registration.

## External 

This endpoint accepts requests from the app and checks for users.

### HTTP Request

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

`GET http://202.164.192.189:3001/pktnew-user.json`

#### Query Parameters

Parameter | Description
--------- | -----------
username | Username / E-Mail address used in DataGene Web / Mobile registration.
password | Password used in DataGene Web / Mobile registration.

<aside class="warning">
Remember — 403 error means username or password is incorrect!
</aside>

# Register new user

## Internal

This endpoint accepts requests from registration forms, fetches IP addresses of DPC's and passes through to external server for processing.

### HTTP Request
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

`GET /app/Settings/asynchttp_register`

#### Query Parameters

Parameter | Description
--------- | -----------
pktoptions_username | E-Mail address for registration
md5password | md5 hashed password for registration
pktupdates_sync | Herd Mask
tech | Farm tech set to first part of email address

## External

This endpoint checks the User is available on the server and returns the DPC number to be used for sensing registration.

### CHECK USER HTTP Request

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

`GET http://202.164.192.189:3001/usercheck.json`

#### Query Parameters

Parameter | Description
--------- | -----------
login | E-Mail address for registration
herds | Herd list user is registering



This 'PocketService' endpoint sends the registration request to the server.

### SEND Registration request

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

`POST http://202.164.192.189:9890`

#### Query Parameters

Parameter | Description
--------- | -----------
Pktoptions_UserName | E-Mail address for registration
Pktoptions_Password | Password for registration
Pktoptions_HerdIDs | Herd list user is registering
Pktoptions_TechNames | Name of tech (email address before '@')
Pktoptions_HTCentre | DPC number
Pktoptions_id | client id generated for device

# Cow List

## General Listings

```javascript
// Framework 7 - DOM 7 getJSON call for data

$$.getJSON('/app/Pktfemale/find_cows', function (json) {
	// generate list to be inserted into DOM here.
});
```

```ruby
  def find_cows
  	# Check if there are cows for the herd
    $pktfemales = Pktfemale.find(:first, :conditions => ["Cows_HerdID =?","#{$HerdID}"])
    # Load options
    @pktoptions = Pktoption.find(:first)
    # Set page for pagination if used (Currently off with @per_page set to 2000 below)
    if @params['page'].to_i == 1
      @page = 0
    else
      @page = @params['page'].to_i
    end
    @per_page = 2000
    # Set Order
    @order = 'Cows_HIONoSorted'

    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_CowStatus!=? ", dq($HerdID), "3"],:select =>['Cows_HIONo','Cows_CowStatus','object'], :page => @page, :per_page => @per_page, :order => @order)  
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_CowStatus!=? ", dq($HerdID), "3"],:select =>['Cows_HIONo','Cows_CowStatus','object'], :page => @page, :per_page => @per_page, :order => @order)  
    end
    # Generate content as JSON and render string for front end to handle
    gen_content = @pktfemales.to_json
    @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
    render :string => gen_content, :layout => false  
  end
```

This generalizes the cow list endpoints.

`GET /app/Pktfemale/find_cows`

`GET /app/Pktfemale/milk_search`

`GET /app/Pktfemale/dry_find`

`GET /app/Pktfemale/heif_search`

`GET /app/Pktfemale/calf_search`


## Search functions

```javascript
//Framework 7 - change view based on search criteria

mainView.router.load({url: '/app/Pktfemale/calving_search?search='+$$('#calvingsearch').val()});
```

```ruby
  def show
    #If List item clicked go straight to record
    if @params['id']
      @pktfemale = Pktfemale.find(@params['id'])
    else
    # If search used, lookup animal
      $search = @params['search']
      if $PrefID == "Y"
        @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus !=? ', dq($HerdID), $search,"3"]) 
      else
        @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus !=? ', dq($HerdID), $search,"3"]) 
      end  
    end
    if @pktfemale
      # If found, fetch event, test and lact details
      $cowlist = true      
      @pktcowevents = Pktcowevent.find(:all, :conditions => ["CowEvents_CowID =?","#{@pktfemale.id}"], :order => 'CowEvents_Date', :orderdir => "DESC")
      @pktcowtests = Pkttest.find(:all, :conditions => ["CowTests_CowID =?","#{@pktfemale.id}"], :select => ['TestDays_Date','CowTests_CowID','CowTests_Yield1','CowTests_Fat','CowTests_FatP','CowTests_Prot','CowTests_ProtP','CowTests_ICCC'], :order => 'TestDays_Date', :orderdir => "DESC")
      @pktlacts = Pktlact.find(:all, :conditions => ["Lacts_CowID =?","#{@pktfemale.id}"], :select => ['Lacts_CowID','Lacts_InitDate','Stats_TotalYield','Stats_TotalFatPerc','Stats_TotalProtPerc','Stats_CurPI'], :order => 'Lacts_InitDate', :orderdir => 'DESC')
      render :action => :show
    else
      # If not found, double check ID param from list item click and fetch event, test and lact details
      @pktfemale = Pktfemale.find(:first, :conditions => {"id" => "#{@params['id']}"})
      $cowlist = true
      @pktcowevents = Pktcowevent.find(:all, :conditions => ["CowEvents_CowID =?","#{@pktfemale.id}"], :order => 'CowEvents_Date', :orderdir => "DESC")
      @pktcowtests = Pkttest.find(:all, :conditions => ["CowTests_CowID =?","#{@pktfemale.id}"], :select => ['TestDays_Date','CowTests_CowID','CowTests_Yield1','CowTests_Fat','CowTests_FatP','CowTests_Prot','CowTests_ProtP','CowTests_ICCC'], :order => 'TestDays_Date', :orderdir => "DESC")
      @pktlacts = Pktlact.find(:all, :conditions => ["Lacts_CowID =?","#{@pktfemale.id}"], :select => ['Lacts_CowID','Lacts_InitDate','Stats_TotalYield','Stats_TotalFatPerc','Stats_TotalProtPerc','Stats_CurPI'], :order => 'Lacts_InitDate', :orderdir => 'DESC')
      render :action => :show
    end
  end
```

This generalizes the search endpoints when typing an animal ID and pressing done.

`GET /app/Pktfemale/show`

Parameter | Description
--------- | -----------
search | Herd Recording number of animal
id | object id of record (Used in list item click)

# Data Entry

## Calving
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/calving_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def calving_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
    
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus != ? AND Cows_CowStatus != ? AND Cows_CowStatus != ?", dq($HerdID), "%#{@params['search']}%", "1","4","3"], :page => @page, :per_page => @per_page, :order => @order)      
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus != ? AND Cows_CowStatus != ? AND Cows_CowStatus != ?", dq($HerdID), "%#{@params['search']}%", "1","4","3"], :page => @page, :per_page => @per_page, :order => @order)  
    end
    gen_content = @pktfemales.to_json
    @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
    render :string => gen_content, :layout => false
  end
```

This endpoint is used to list Cows to Calve (Dry cows and Heifers).

`GET /app/Pktfemale/calving_index_list`

###  List Item Click
```ruby
  def calving_show
    @pktfemale = Pktfemale.find(@params['id'])
    @CalvDate = Date.today
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    # Calving must be after #{@minEDate} #{@minEDateReason}
    @minEDate = Date.today - 1000;
    @minEDateReason = "as it is too long ago"
    if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') + 540 > @minEDate)
      @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') + 540
      @minEDateReason = "as is too young to calve"
    end
    if !@pktfemale.Cows_LastDryOff.nil?
      if @pktfemale.Cows_LastDryOff != ''
        if (Date.strptime(@pktfemale.Cows_LastDryOff, '%Y-%m-%d') + 1 > @minEDate)
          @minEDate = Date.strptime(@pktfemale.Cows_LastDryOff, '%Y-%m-%d') + 1
          @minEDateReason = "it's last dryoff"
        end
      end
    end
    if !@pktfemale.Cows_LastCalvDate.nil? 
      if @pktfemale.Cows_LastCalvDate != ''
        if (Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d') + 200 > @minEDate)
          @minEDate = Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d') + 200
          @minEDateReason = "at least 200 days after last calving"
        end
      end
    end

    if @minEDate > @CalvDate
      WebView.executeJavascript("myApp.alert('Calving must be after #{@minEDate} #{@minEDateReason}','Error!')")
    else
      render :action => :calving_edit
    end
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Calving screen.

`GET /app/Pktfemale/calving_show`


Parameter | Description
--------- | -----------
id | object id of record (Used in list item click)

### Search Animal
```ruby
  def calving_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']

       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus != ? AND Cows_CowStatus != ? ', dq($HerdID), "#{@params['search']}", "1","4"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif ["0","2"].include?(@pktfemale.Cows_CowStatus)      

            @CalvDate = Date.today

          # Calving must be after #{@minEDate} #{@minEDateReason}
          @minEDate = Date.today - 1000;

          if @minEDate > Date.today
            WebView.executeJavascript("Calving must be after #{@minEDate} #{@minEDateReason}","Calving Date")
            #render :action => :calving_index   
          elsif !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > (Date.today - 520))
            WebView.executeJavascript("The cow is to young to calve!","Too Young")
            #render :action => :calving_index
          else
            render :action => :calving_edit
          end
        end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "1"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
         #redirect :action => :calving_index
       elsif ["0","2"].include?(@pktfemale.Cows_CowStatus)      
            @CalvDate = Date.today
 
          # Calving must be after #{@minEDate} #{@minEDateReason}
          @minEDate = Date.today - 1000;

          if @minEDate > Date.today
            WebView.executeJavascript("Calving must be after #{@minEDate} #{@minEDateReason}","Calving Date")
            #render :action => :calving_index   
          elsif !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > (Date.today - 520))
            WebView.executeJavascript("The cow is to young to calve!","Too Young")
            #render :action => :calving_index
          end
         render :action => :calving_edit
       end
    end
  end
```

This endpoint is used to find the specified animal and navigate to the Calving screen.

`GET /app/Pktfemale/calving_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postCalveCowData() 
{
  // Validate fields
  if (CalvingValidate())
  {
    // Convert formdata to JSON
    var formData = myApp.formToJSON('#calvingForm');
    // Serialize JSON object
    var data = $$.serializeObject(formData);
    // Post to controller action
    $$.post('/app/Pktfemale/calving_update', data, function (data) {
	});
	// Navigate back to list when POST complete
	mainView.router.back({url:'/app/Pktfemale/calving_index', force: true, ignoreCache:true});
  }
}
```

```ruby
  def calving_update
    require_source "Report"
    # Create Event record
    @pktevent = Pktcowevent.create(@params['pktcowevent'])
    # Create Update record
    @pktupdate = Pktupdate.create(@params['pktupdate'])
    @pktfemale = Pktfemale.find(@params['id'])    
    $date = @params['sampleDate']
    # Store ID, Date and Status in arbitary table for reversal
    db = ::Rho::RHO.get_src_db('Report')
    db.start_transaction
    begin
      mytest = Start.new
      mytest.ID = "#{@params['id']}"    
      mytest.Date = "#{@params['pktupdate']['Pktupdates_EDate']}"
      mytest.Status = "#{@pktfemale.Cows_CowStatus}"   
      mytest.save
      db.commit
      rescue
      db.rollback
    end
    #Update animal details
    @pktfemale.update_attributes("Cows_CowStatus" => "1", "Cows_LastCalvDate" => "#{$date}") if @pktfemale
  end
```

This endpoint is used to store the details of the calving in the local database

`POST /app/Pktfemale/calving_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
pktfemale[Cows_CowStatus] | 1 |Status of cow
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | C |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Calving

## Dry Off
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/dry_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def dry_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
    
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus=? ", dq($HerdID), "%#{@params['search']}%", "1"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false    

    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus=? ", dq($HerdID), "%#{@params['search']}%", "1"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false    
    end
  end
```
This endpoint is used to list Cows to Dry Off.

`GET /app/Pktfemale/dry_index_list`

### List Item Click

```ruby
  def dry_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @mastitis = Pktcupboard.find(:all, :conditions => ["Cupboards_EGroup like ?", "15"])
    @pktfemale = Pktfemale.find(@params['id'])
    
    @minEDate = Date.today - 1000;
    @minEDateReason = "it is too long ago"
    if !@pktfemale.Cows_LastTest.nil? and (Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d') > @minEDate)
        @minEDate = Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d')
        @minEDateReason = "it's last test date"
    end
    if !@pktfemale.Cows_LastCalvDate.nil? and (Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d') > @minEDate)
       @minEDate = Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d')
       @minEDateReason = "it's last calving"
    end
    render :action => :dry_new
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Dry Off screen.

`GET /app/Pktfemale/dry_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def dry_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus=? ', dq($HerdID), "#{@params['search']}", "1"])  
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         @mastitis = Pktcupboard.find(:all, :conditions => ["Cupboards_EGroup like ?", "15"])
         # Dry off must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it is too long ago"
         if !@pktfemale.Cows_LastTest.nil? and (Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d') > @minEDate)
             @minEDate = Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d')
             @minEDateReason = "it's last test date"
         end
         if !@pktfemale.Cows_LastCalvDate.nil? and (Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d') > @minEDate)
            @minEDate = Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d')
            @minEDateReason = "it's last calving"
         end
         render :action => :dry_new
       end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus=? ', dq($HerdID), $search, "1"])  
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         @mastitis = Pktcupboard.find(:all, :conditions => ["Cupboards_EGroup like ?", "15"])
         # Dry off must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it is too long ago"
         if !@pktfemale.Cows_LastTest.nil? and (Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d') > @minEDate)
             @minEDate = Date.strptime(@pktfemale.Cows_LastTest, '%Y-%m-%d')
             @minEDateReason = "it's last test date"
         end
         if !@pktfemale.Cows_LastCalvDate.nil? and (Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d') > @minEDate)
            @minEDate = Date.strptime(@pktfemale.Cows_LastCalvDate, '%Y-%m-%d')
            @minEDateReason = "it's last calving"
         end
         render :action => :dry_new
       end
    end
  end
```

This endpoint is used to find the specified animal and navigate to the Dry Off screen.

`GET /app/Pktfemale/dry_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postDryOffCowData() 
{
  if (dryOffCowValidate())
  {  
      var formData = myApp.formToJSON('#dryoffCowForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/dry_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/dry_index', force: true, ignoreCache:true});    
  }   
}
```

```ruby
  def dry_update
    $date = "#{@params['dateDry1']}"
    $DryDrug = @params['drydrug']
    $TeatSeal = "#{@params['teatsealevent']}"
    #@date = Date.strptime("#{$date}", '%d/%m/%Y')
    #@date = @date.strftime('%Y-%m-%d')
    @pktfemale = Pktfemale.find(@params['id'])
    @pktupdateID = rand(10**12-10)+3
    @pktfemale.update_attributes({"Cows_LastDryOff" => "#{@params['pktupdate']['Pktupdates_EDate']}", "Cows_CowStatus" => "2", "Cows_LastDryOffCode" => "#{@params['Cows_LastryOffCode']}"})
    @pktupdate = Pktupdate.create({
      "Pktupdates_ID" => "#{@pktupdateID}",
      "Pktupdates_HerdID" => "#{$HerdID}",
      "Pktupdates_Entered" => "#{@params['pktupdate']['Pktupdates_Entered']}",
      "Pktupdates_CowNo" => "#{@pktfemale.Cows_HIONo}",
      "Pktupdates_Process" => "D",
      "Pktupdates_ECode" => "#{@params['Cows_LastDryOffCode']}",
      "Pktupdates_EDate" => "#{@params['pktupdate']['Pktupdates_EDate']}",
      "Pktupdates_Status" => "N",
      "Pktupdates_AnimalID" => "#{@params['pktupdate']['Pktupdates_AnimalID']}",
      "Pktupdates_DataString" => "Treatment: #{@params['Events_ECode']}"
    })
    
    @pktcowevents = Pktcowevent.create({
      "id" => "#{@pktupdateID}",
      "CowEvents_CowID" => "#{@params['id']}",
      "CowEvents_HerdID" => "#{$HerdID}",
      "CowEvents_Date" => "#{$date}",
      "Events_ECode" => "#{@params['Cows_LastDryOffCode']}"
    })

    @pktcowevents = Pktcowevent.create({
      "id" => "#{@pktupdateID}",
      "CowEvents_CowID" => "#{@params['id']}",
      "CowEvents_HerdID" => "#{$HerdID}",
      "CowEvents_Date" => "#{@params['pktupdate']['Pktupdates_EDate']}",
      "Events_ECode" => "#{@params['Events_ECode']}"
    })
    
    if $TeatSeal == 'on'  
      @PktupdateID = pktupdateCreateID()    
      @pktcowevents = Pktcowevent.create({
        "id" => "#{@pktupdateID}",
        "CowEvents_CowID" => "#{@params['id']}",
        "CowEvents_HerdID" => "#{$HerdID}",
        "CowEvents_Date" => "#{@params['pktupdate']['Pktupdates_EDate']}",
        "Events_ECode" => "TEATSEAL"
      })

      @pktupdate = Pktupdate.create({
        "Pktupdates_ID" => "#{@PktupdateID}",
        "Pktupdates_HerdID" => "#{$HerdID}",
        "Pktupdates_Entered" => "#{@params['pktupdate']['Pktupdates_Entered']}",
        "Pktupdates_CowNo" => "#{@pktfemale.Cows_HIONo}",
        "Pktupdates_Process" => "T",
        "Pktupdates_ECode" => "TEATSEAL",
        "Pktupdates_EDate" => "#{@params['pktupdate']['Pktupdates_EDate']}",
        "Pktupdates_Status" => "N",
        "Pktupdates_AnimalID" => "#{@params['pktupdate']['Pktupdates_AnimalID']}"
      })      
    end  
  end
```

This endpoint is used to store the details of the dry off in the local database

`POST /app/Pktfemale/dry_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
teatsealevent | false | Whether teatseal has been used or not
techID | TechID | Name of farm tech
dateDry1 | Now | Selected date for dry off
pktfemale[Cows_CowStatus] | 2 |Status of cow
pktfemale[Cows_LastDryOffCode] | NULL | Dry off code
pktcowevent[id] | NULL | Generated ID for cowevents
pktcowevent[CowEvents_CowID] | NatID | National ID of animal
pktcowevent[CowEvents_HerdID] | Herd |Herd Mask
pktcowevent[CowEvents_Date] | Today | Event Date
pktcowevent[Events_ECode] | NULL | Event Code
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | D |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Dry Off Event

## Health Events
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/event_index_list', function (json) {
  // Insert list to DOM here
});
```

```ruby
  def event_index_list
    if @params['page'].to_i == 1
       @page = 0
    else
       @page = @params['page'].to_i
    end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
    
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{@params['search']}%", "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false    
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{@params['search']}%", "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false 
    end
  end
```
This endpoint is used to list cows to add Health Events to.

`GET /app/Pktfemale/event_index_list`

### List Item Click

```ruby
  def event_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @pktfemale = Pktfemale.find(@params['id'])
    # Event date must be after #{@minEDate} #{minEDateReason}
    @minEDate = Date.today - 1000;
    @minEDateReason = "it's too long ago"
    if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
      @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
      @minEDateReason = "it's birth date"
    end   
    @repro = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "1"], :order => 'Events_Name')
    @udder = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "2"], :order => 'Events_Name')
    @legs = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "3"], :order => 'Events_Name')
    @disease = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "4"], :order => 'Events_Name')
    @mating = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "0"], :order => 'Events_Name')    
    render :action => :event_new
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Add Health Event screen.

`GET /app/Pktfemale/event_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def event_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus!=? ', dq($HerdID), "#{@params['search']}", "3"]) 
      if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
      elsif @pktfemale
        # Event date must be after #{@minEDate} #{minEDateReason}
        @minEDate = Date.today - 1000;
        @minEDateReason = "it's too long ago"
        if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
          @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
          @minEDateReason = "it's birth date"
        end   
        @repro = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "1"], :order => 'Events_Name')
        @udder = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "2"], :order => 'Events_Name')
        @legs = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "3"], :order => 'Events_Name')
        @disease = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "4"], :order => 'Events_Name')
        @mating = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "0"], :order => 'Events_Name')    
        render :action => :event_new
      end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         # Event date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end   
         @repro = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "1"], :order => 'Events_Name')
         @udder = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "2"], :order => 'Events_Name')
         @legs = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "3"], :order => 'Events_Name')
         @disease = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "4"], :order => 'Events_Name')
         @mating = Pktevent.find(:all, :conditions => ["Events_EGroup like ?", "0"], :order => 'Events_Name')    
         render :action => :event_new
       end
    end
  end
```

This endpoint is used to find the specified animal and navigate to the Add Health Event screen.

`GET /app/Pktfemale/event_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postCowEventData() 
{
  if (healthValidate())
  {
    var eventChosen = document.getElementById('udder').value;
      var formData = myApp.formToJSON('#healthForm');
      var data = $$.serializeObject(formData);
    // Check for Mastitis event code
    if (eventChosen.indexOf('MCT') == 0|| eventChosen.indexOf('MCA') == 0|| eventChosen.indexOf('STREP') == 0|| eventChosen.indexOf('STAPH') == 0|| eventChosen.indexOf('UBER') == 0|| eventChosen.indexOf('COLI') == 0|| eventChosen.indexOf('MCNT') == 0) {
        myApp.modal({
          title:  'Mastitis Event',
          text: 'You are entering a Mastitis event, would you like to enter an treatment for this?',
          buttons: [
            {
              text: 'No',
              onClick: function() {
                    $$.post('/app/Pktfemale/create_event', data, function (returneddata) {
              if (returneddata == 'true') {
                myApp.alert("A "+ECodeValue+" Event on "+document.getElementById('dateEvent1').value+" at this time already exists for cow: "+ document.getElementById('animalHIONo').value,'Event exists!');
              } else {
                mainView.router.back({url:'/app/Pktfemale/event_index', force: true, ignoreCache:true});      
              }
            }); 
              }
            },
            {
              text: 'Yes',
              onClick: function() {
                $$.post('/app/Pktfemale/create_event', data, function (returneddata) {
                if (returneddata == 'true') {
                  myApp.alert("A "+ECodeValue+" Event on "+document.getElementById('dateEvent1').value+" at this time already exists for cow: "+ document.getElementById('animalHIONo').value,'Event exists!');
              } else {
                mainView.router.back({url:'/app/Pktfemale/treatment_show?id='+$$('#cowObject').val(), force: true, ignoreCache:true});      
              }
            });    
              }
            },
          ]
        })
    } else {
        $$.post('/app/Pktfemale/create_event', data, function (returneddata) {
        if (returneddata == 'true') {
          myApp.alert("A "+ECodeValue+" Event on "+document.getElementById('dateEvent1').value+" at this time already exists for cow: "+ document.getElementById('animalHIONo').value,'Event exists!');
        } else {
          mainView.router.back({url:'/app/Pktfemale/event_index', force: true, ignoreCache:true});      
        }
      }); 
    }
  }
}
```

```ruby
  def create_event
    $date = "#{@params['sampleDate']}"
    # Check if event exists
    @eventexists = Pktcowevent.find(:first, :conditions => ["CowEvents_CowID=? AND CowEvents_HerdID=? AND CowEvents_Date=? AND Events_ECode=? AND CowEvents_RXTime=?","#{@params['pktcowevent']['CowEvents_CowID']}","#{@params['pktcowevent']['CowEvents_HerdID']}","#{@params['pktupdate']['Pktupdates_EDate']}","#{@params['pktcowevent']['Events_ECode']}","#{@params['pktcowevent']['CowEvents_RxTime']}"])
    # If exists return true to UI
    if @eventexists
      @pktfemale = Pktfemale.find(:first, :conditions => ["id = ?", "#{@params['pktcowevent']['CowEvents_CowID']}"])
      render :string => 'true'
    else
      #create event
      @pktfemale = Pktfemale.find(:first, :conditions => ["id = ?", "#{@params['pktcowevent']['CowEvents_CowID']}"]) 
      @pktcowevent = Pktcowevent.create(@params['pktcowevent'])
      @pktupdateID = rand(10**12-10)+3
      @pktupdate = Pktupdate.create({
        "Pktupdates_ID" => "#{@pktupdateID}",
        "Pktupdates_HerdID" => "#{$HerdID}",
        "Pktupdates_Entered" => "#{@params['pktupdate']['Pktupdates_Entered']}",
        "Pktupdates_CowNo" => "#{@pktfemale.Cows_HIONo}",
        "Pktupdates_Process" => "H",
        "Pktupdates_ECode" => "#{@params['pktupdate']['Pktupdates_ECode']}",
        "Pktupdates_EDate" => "#{@params['pktupdate']['Pktupdates_EDate']}",
        "Pktupdates_Status" => "N",
        "Pktupdates_AnimalID" => "#{@params['pktupdate']['Pktupdates_AnimalID']}",
        "Pktupdates_DataString" => "#{@params['pktupdate']['Pktupdates_DataString']}"
      })
    end
  end
```

This endpoint is used to store the details of the cow event the local database

`POST /app/Pktfemale/create_event`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
techID | TechID | Name of farm tech
pktfemale[Cows_LastDryOffCode] | NULL | Dry off code
pktcowevent[id] | NULL | Generated ID for cowevents
pktcowevent[CowEvents_CowID] | NatID | National ID of animal
pktcowevent[CowEvents_HerdID] | Herd |Herd Mask
pktcowevent[CowEvents_Date] | Today | Event Date
pktcowevent[Events_ECode] | NULL | Event Code
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | H |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Health Event

## Mating
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/mating_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def mating_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
     if $PrefID =="Y"
       @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{$search}%", "3"],:select =>['Cows_HRID','Cows_CowStatus','object','Cows_HerdGroup'], :page => @page, :per_page => @per_page, :order => @order)  
     else
       @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{$search}%", "3"],:select =>['Cows_HIONo','Cows_CowStatus','object','Cows_HerdGroup'], :page => @page, :per_page => @per_page, :order => @order)  
     end
     gen_content = @pktfemales.to_json
     @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
     render :string => gen_content, :layout => false
  end
```
This endpoint is used to list cows to add join.

`GET /app/Pktfemale/mating_index_list`

### List Item Click

```ruby
  def mating_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @pktfemale = Pktfemale.find(@params['id'])
    @pktbteams = Pktbteam.find(:all, :order => 'BTeams_Nasis2')
    # Mating date must be after #{@minEDate} #{minEDateReason}
    @minEDate = Date.today - 1000;
    @minEDateReason = "it's too long ago"
    if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
      @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
      @minEDateReason = "it's birth date"
    end   
    render :action => :mating_edit
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Add Mating screen.

`GET /app/Pktfemale/mating_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def mating_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus!=? ', dq($HerdID), "#{$search}", "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       else
         @pktbteams = Pktbteam.find(:all, :order => 'BTeams_Nasis2')
         # Mating date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end   
         render :action => :mating_edit
       end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "3"]) 
       if !@pktfemale
         okBox("Cow #{$search} does not exist in the database for herd #{$HerdID}.","/app/Pktfemale/mating_index","Not Found")
       else
         @pktbteams = Pktbteam.find(:all, :order => 'BTeams_Nasis2')
         # Mating date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end   
         render :action => :mating_edit
       end
    end
  end
```

This endpoint is used to find the specified animal and navigate to the Add Mating screen.

`GET /app/Pktfemale/mating_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postMatingData() 
{
  if (matingValidate())
  {
      var formData = myApp.formToJSON('#matingForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/mating_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/mating_index', force: true, ignoreCache:true});  
  }
}
```

```ruby
  def mating_update
    $date = "#{@params['dateMate1']}"
    $ECode = "#{@params['pktcowevent']['Events_ECode]']}"
    $LastBull = "#{@params['pktcowevent']['CowEvents_Sire]']}"
    @pktcowevent = Pktcowevent.create(@params['pktcowevent'])
    @pktfemale = Pktfemale.find(@params['id'])
    @pktupdate = Pktupdate.create(@params['pktupdate'])
    @pktfemale.update_attributes(@params['pktfemale']) if @pktfemale
  end
```

This endpoint is used to store the details of the mating in the local database

`POST /app/Pktfemale/mating_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
techID | TechID | Name of farm tech
dateMate1 | Today | Date of joining
pktfemale[Cows_DueDate] | NULL | Calculated Due Date
pktfemale[Cows_DueDryOff] | NULL | Calculated Dry Off Date
pktfemale[Cows_LastServ] | NOW | Date of Last joining
pktcowevent[id] | NULL | Generated ID for cowevents
pktcowevent[CowEvents_CowID] | NatID | National ID of animal
pktcowevent[CowEvents_HerdID] | Herd |Herd Mask
pktcowevent[CowEvents_Date] | Today | Event Date
pktcowevent[Events_ECode] | NULL | Event Code
cowevent[CowEvents_Sire] | NULL | Nasis2 ID of Bull joined to
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | M |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Mating Event


## Preg Test
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/pregtest_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def pregtest_index_list
    if @params['page'].to_i == 1
      @page = 0
    else
      @page = @params['page'].to_i
    end
    @per_page = 2000
    @order = 'Cows_HIONoSorted'
    $search = @params['search']
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{$search}%", "3"],:select =>['Cows_HRID','Cows_CowStatus','object','Cows_HerdGroup'], :page => @page, :per_page => @per_page, :order => @order)  
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{$search}%", "3"],:select =>['Cows_HIONo','Cows_CowStatus','object','Cows_HerdGroup'], :page => @page, :per_page => @per_page, :order => @order)  
    end
    gen_content = @pktfemales.to_json
    @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
    render :string => gen_content, :layout => false  
  end
```
This endpoint is used to list cows to pregtest.

`GET /app/Pktfemale/pregtest_index_list`

### List Item Click

```ruby
  def pregtest_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @pktfemale = Pktfemale.find(@params['id'])
    # Pregnancy test date must be after #{@minEDate} #{minEDateReason}
    @minEDate = Date.today - 1000;
    @minEDateReason = "it's too long ago"
    if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
      @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
      @minEDateReason = "it's birth date"
    end
    render :action => :pregtest_new 
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Add Preg test screen.

`GET /app/Pktfemale/pregtest_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def pregtest_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
      $search = @params['search']
      @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus!=? ', dq($HerdID), "#{@params['search']}", "3"])  
      if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
      else 
          # Pregnancy test date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end
         render :action => :pregtest_new
      end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       else
           # Pregnancy test date must be after #{@minEDate} #{minEDateReason}
          @minEDate = Date.today - 1000;
          @minEDateReason = "it's too long ago"
          if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
            @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
            @minEDateReason = "it's birth date"
          end
         render :action => :pregtest_new
       end
    end      
  end
```

This endpoint is used to find the specified animal and navigate to the Add Pregtest screen.

`GET /app/Pktfemale/pregtest_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postPDData() 
{
  //return pregTestValidate();
  if (pregTestValidate())
  {
      var formData = myApp.formToJSON('#pregTestForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/preg_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/pregtest_index', force: true, ignoreCache:true}); 
  }
}
```

```ruby
  def preg_update
    $date = "#{@params['datePD1']}"
    @pktfemale = Pktfemale.find(@params['id'])
    @pktcowevent = Pktcowevent.create(@params['pktcowevent'])
    @pktstart = Start.create(@params['start'])
    @pktupdate = Pktupdate.create(@params['pktupdate'])
    @pktfemale.update_attributes(@params['pktfemale']) if @pktfemale
    #WebView.navigate("/app/Pktfemale/pregtest_index")
  end
```

This endpoint is used to store the details of the pregtest in the local database

`POST /app/Pktfemale/preg_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
datePD1 | Today | Date of preg test
pktfemale[Cows_DueDate] | NULL | Calculated Due Date
pktcowevent[id] | NULL | Generated ID for cowevents
pktcowevent[CowEvents_CowID] | NatID | National ID of animal
pktcowevent[CowEvents_HerdID] | Herd |Herd Mask
pktcowevent[CowEvents_Date] | Today | Event Date
pktcowevent[Events_ECode] | NULL | Event Code
cowevent[CowEvents_Sire] | NULL | Nasis2 ID of Bull joined to
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | M |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Preg Test Event

## Sell / Cull
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/terminate_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def terminate_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
    
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{@params['search']}%", "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false     
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{@params['search']}%", "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false
    end
  end 
```
This endpoint is used to list cows to terminate.

`GET /app/Pktfemale/terminate_index_list`

### List Item Click

```ruby
  def terminate_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @pktfemale = Pktfemale.find(@params['id'])
    render :action => :terminate_new
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Terminate screen.

`GET /app/Pktfemale/pregtest_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def terminate_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus!=? ', dq($HerdID), "#{@params['search']}", "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         render :action => :terminate_new
       end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         render :action => :terminate_new
       end
    end   
  end  
```

This endpoint is used to find the specified animal and navigate to the Terminate Animal screen.

`GET /app/Pktfemale/terminate_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postTermData() 
{
  if (TerminateValidate())
  {
      var formData = myApp.formToJSON('#terminateForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/term_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/terminate_index', force: true, ignoreCache:true});      
  }
}
```

```ruby
  def term_update
    @pktfemale = Pktfemale.find(@params['id'])
    $date = @params['dateTerm1']
    
    @data = Start.create(
      {"ID" => "#{@params['id']}", "Date" => "#{@params['pktupdate']['Pktupdates_EDate']}", "Status" => "#{@pktfemale.Cows_CowStatus}"}
    )
    
    @pktupdateID = rand(10**12-10)+3
    @pktfemale.update_attributes({"Cows_TermDate" => "#{@params['pktupdate']['Pktupdates_ECode']}", "Cows_CowStatus" => "3", "Cows_ExitCode" => "#{@params['pktupdate']['Pktupdates_ECode']}"})
    @pktupdate = Pktupdate.create({
      "Pktupdates_ID" => "#{@pktupdateID}",
      "Pktupdates_HerdID" => "#{$HerdID}",
      "Pktupdates_Entered" => "#{@params['pktupdate']['Pktupdates_Entered']}",
      "Pktupdates_CowNo" => "#{@pktfemale.Cows_HIONo}",
      "Pktupdates_Process" => "S",
      "Pktupdates_ECode" => "#{@params['pktupdate']['Pktupdates_ECode']}",
      "Pktupdates_EDate" => "#{@params['pktupdate']['Pktupdates_EDate']}",
      "Pktupdates_Status" => "N",
      "Pktupdates_AnimalID" => "#{@params['pktupdate']['Pktupdates_AnimalID']}",
      "Pktupdates_DataString" => "#{@params['pktupdate']['Pktupdates_DataString']}"
    })    
   
    #WebView.navigate("/app/Pktfemale/terminate_index")
  end
```

This endpoint is used to store the details of the termination in the local database

`POST /app/Pktfemale/term_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
dateTerm1 | Today | Date of preg test
pktfemale[Cows_CowStatus] | 3 | Terminated status id
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | S |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Termination Event

## Treatment
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/treatment_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def treatment_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search'] 
   
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus!=? ", dq($HerdID), "%#{@params['search']}%", "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false     
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_CowStatus!=? ", dq($HerdID), "3"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false 
    end
  end 
```
This endpoint is used to list cows to add treatments to.

`GET /app/Pktfemale/treatment_index_list`

### List Item Click

```ruby
  def treatment_show
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    @pktfemale = Pktfemale.find(@params['id'])
    # Treatment date must be after #{@minEDate} #{minEDateReason}
    @minEDate = Date.today - 1000;
    @minEDateReason = "it's too long ago"
    if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
      @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
      @minEDateReason = "it's birth date"
    end
    loadTreatments()
    render :action => :treatment_new
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Add Treatment screen.

`GET /app/Pktfemale/treatment_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def treatment_search
    if !$date
      $date = Date.today.strftime('%Y-%m-%d')
    end
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND Cows_CowStatus!=? ', dq($HerdID), "#{@params['search']}", "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         # Treatment date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end
         loadTreatments()
         render :action => :treatment_new
       end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND Cows_CowStatus!=? ', dq($HerdID), $search, "3"]) 
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif @pktfemale
         # Treatment date must be after #{@minEDate} #{minEDateReason}
         @minEDate = Date.today - 1000;
         @minEDateReason = "it's too long ago"
         if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
           @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
           @minEDateReason = "it's birth date"
         end
         loadTreatments()
         render :action => :treatment_new
       end
    end   
  end 
```

This endpoint is used to find the specified animal and navigate to the Add Treatment screen.

`GET /app/Pktfemale/treatment_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postTermData() 
{
  if (TerminateValidate())
  {
      var formData = myApp.formToJSON('#terminateForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/term_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/terminate_index', force: true, ignoreCache:true});      
  }
}
```

```ruby
  def treat_create_event
    @eventexists = Pktcowevent.find(:first, :conditions => ["CowEvents_CowID=? AND CowEvents_HerdID=? AND CowEvents_Date=? AND Events_ECode=? AND CowEvents_RXTime=? AND Events_NeedLoc=?","#{@params['pktcowevent']['CowEvents_CowID']}","#{@params['pktcowevent']['CowEvents_HerdID']}","#{@params['pktupdate']['Pktupdates_EDate']}","#{@params['pktcowevent']['Events_ECode']}","#{@params['pktcowevent']['CowEvents_RxTime']}","#{@params['pktcowevent']['Events_NeedLoc']}"])
    if @eventexists
      @pktfemale = Pktfemale.find(@params['id'])
      #alertBox("A #{@params['pktupdate']['Pktupdates_ECode']} Event on #{$date} at this time and location already exists for cow: #{@pktfemale.Cows_HIONo}.",'Duplicate Event')
      #WebView.navigate("/app/Pktfemale/treatment_show?id=#{@pktfemale.object}")
      render :string => 'true'
    else
      $date = "#{@params['sampleDate']}"
      @pktfemale = Pktfemale.find(@params['id'])
      @pktupdateID = rand(10**12-10)+3
      @pktcowevent = Pktcowevent.create(@params['pktcowevent'])
      @pktupdate = Pktupdate.create({
        "Pktupdates_ID" => "#{@pktupdateID}",
        "Pktupdates_HerdID" => "#{$HerdID}",
        "Pktupdates_Entered" => "#{@params['pktupdate']['Pktupdates_Entered']}",
        "Pktupdates_CowNo" => "#{@pktfemale.Cows_HIONo}",
        "Pktupdates_Process" => "T",
        "Pktupdates_ECode" => "#{@params['pktupdate']['Pktupdates_ECode']}",
        "Pktupdates_EDate" => "#{@params['pktupdate']['Pktupdates_EDate']}",
        "Pktupdates_Status" => "N",
        "Pktupdates_AnimalID" => "#{@params['pktupdate']['Pktupdates_AnimalID']}",
        "Pktupdates_DataString" => "#{@params['pktupdate']['Pktupdates_DataString']}"
      })
    end    
    #WebView.navigate("/app/Pktfemale/treatment_index")
  end
```

This endpoint is used to store the details of the treatment in the local database

`POST /app/Pktfemale/treat_create_event`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
techID | TechID | Name of farm tech
pktcowevent[id] | NULL | Generated ID for cowevents
pktcowevent[CowEvents_CowID] | NatID | National ID of animal
pktcowevent[CowEvents_HerdID] | Herd |Herd Mask
pktcowevent[CowEvents_Date] | Today | Event Date
pktcowevent[Events_ECode] | NULL | Event Code
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | T |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of Treatment Event

## Workabilities
### List Animals

```javascript
$$.getJSON('/app/Pktfemale/workability_index_list', function (json) {
	// Insert list to DOM here
});
```

```ruby
  def workability_index_list
    if @params['page'].to_i == 1
       @page = 0
     else
       @page = @params['page'].to_i
     end
     @per_page = 2000
     @order = 'Cows_HIONoSorted'
     $search = @params['search']
           
    if $PrefID =="Y"
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HRID like ? AND Cows_CowStatus NOT IN ('0','4','3') ", dq($HerdID), "%#{@params['search']}%"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false     
    else
      @pktfemales = Pktfemale.paginate(:conditions => ["Cows_HerdID like ? AND Cows_HIONo like ? AND Cows_CowStatus NOT IN ('0','4','3') ", dq($HerdID), "%#{@params['search']}%"], :page => @page, :per_page => @per_page, :order => @order)  
      gen_content = @pktfemales.to_json
      @response["headers"]["Content-Type"] = "application/json; charset=utf-8"  
      render :string => gen_content, :layout => false
    end
  end
```
This endpoint is used to list cows to score workabilities.

`GET /app/Pktfemale/workability_index_list`

### List Item Click

```ruby
  def workability_show
    @pktfemale = Pktfemale.find(@params['id'])
       # Pregnancy test date must be after #{@minEDate} #{minEDateReason}
      @minEDate = Date.today - 1000;
      @minEDateReason = "it's too long ago"
      if !@pktfemale.Cows_Birth.nil? and (Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d') > @minEDate)
        @minEDate = Date.strptime(@pktfemale.Cows_Birth, '%Y-%m-%d')
        @minEDateReason = "it's birth date"
      end
       render :action => :workability_new 
  end
```

This endpoint is used to find the specified animal from the list and navigate to the Workability screen.

`GET /app/Pktfemale/workability_show`

Parameter | Description
--------- | -----------
id | object id of record

### Search Animal

```ruby
  def workability_search
    if $PrefID == "Y"
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HRID=? AND (Cows_CowStatus!=? OR Cows_CowStatus !=?) AND Cows_CowStatus !=? ', dq($HerdID), "#{@params['search']}", "0", "4","3"])  
       if !@pktfemale
         WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif ["1","2"].include?(@pktfemale.Cows_CowStatus)   

        # Workability date must be after #{@minEDate} #{minEDateReason}
        @minEDate = Date.today - 60;
        @minEDateReason = "it's too long ago"
        render :action => :workability_new
       end
    else
       $search = @params['search']
       @pktfemale = Pktfemale.find(:first, :conditions => ['Cows_HerdID=? AND Cows_HIONo=? AND (Cows_CowStatus!=? OR Cows_CowStatus !=?) AND Cows_CowStatus !=? ', dq($HerdID), $search,"0", "4","3"]) 
       if !@pktfemale
        WebView.executeJavascript("myApp.alert('Cow #{$search} does not exist in the database for herd #{$HerdID}.','Not Found');")
       elsif ["1","2"].include?(@pktfemale.Cows_CowStatus) 

        # Workability date must be after #{@minEDate} #{minEDateReason}
        @minEDate = Date.today - 60;
        @minEDateReason = "it's too long ago"
        render :action => :workability_new
       end
    end
  end
```

This endpoint is used to find the specified animal and navigate to the Workability screen.

`GET /app/Pktfemale/workability_search`


Parameter | Description
--------- | -----------
search | Herd Recording number of animal

### POST to local database

```javascript
function postTermData() 
{
  if (TerminateValidate())
  {
      var formData = myApp.formToJSON('#terminateForm');

      var data = $$.serializeObject(formData);

      $$.post('/app/Pktfemale/term_update', data, function (data) {
    });
    mainView.router.back({url:'/app/Pktfemale/terminate_index', force: true, ignoreCache:true});      
  }
}
```

```ruby
  def work_update
    @pktfemale = Pktfemale.find(@params['id'])
    @pktupdate = Pktupdate.create(@params['pktupdate'])
    @pktfemale.update_attributes(@params['pktfemale']) if @pktfemale
    #WebView.navigate("/app/Pktfemale/workability_index")
  end
```

This endpoint is used to store the details of the workability scoring in the local database

`POST /app/Pktfemale/work_update`

#### Query Parameters
Parameter | Default | Description
--------- | ------- | -----------
id | object field | object id of animal record
pktfemale[Cows_WorksDate] | Today | Date of workability scoring
pktupdate[Pktupdates_ID] | NULL | Generated ID for pktupdates record
pktupdate[Pktupdates_Entered] | Now | DateTime stamp record is being created
pktupdate[Pktupdates_HerdID] | Herd |Herd Mask
pktupdate[Pktupdates_Process] | W |Process being crated
pktupdate[Pktupdates_CowNo] | Cow | Cow herd recording number
pktupdate[Pktupdates_EDate] | Today | Event Date
pktupdate[Pktupdates_ECode] | NULL | Event Code
pktupdate[Pktupdates_Status] | N | Status of record
pktupdate[Pktupdates_AnimalID] | Nat ID | National ID of animal
pktupdate[Pktupdates_DataString] | NULL | Details of workability scoring