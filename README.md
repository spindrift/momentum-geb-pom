momentum-geb-pom
================

Momentum User tests with Geb using inline scripting and Spock tests via gradle, extending the Page Object Model.

Includes the gradle wrapper if you don't have gradle installed.
For all gradle executions below use the wrapper:
	./gradlew [task...] [option ...]
or if you have gradle installed:
	gradle [task...] [option ...]
	
If using Eclipse the eclipse plugin is included. After checkout run the following then import the project into Eclipse.

	gradle eclipse
	
The tests assume you have Momentum Storefront running on localhost on port 8080 and the BCC running on port 8180, either local on your host
or in a VM with port forwarding.

This project is demonstrating 3 things

1. How to use Geb to screenscrape and use the results to generate our Page classes
2. Run Spock tests with some of the generated Page classes
3. Run equivalent scripts against the generated Page classes

Step 1
------
The first step involves generating the actual classes themselves by using Geb again to do some screen-scraping to generate a list of current URLs 
from a site in the BCC. Then we hit the storefront with each url to get the page title and finally that we have all the names, urls and titles we 
can generate all the page classes.
To make things faster there is a doNothing task so that scripts can be run without invoking a task that does anything, but the default task is testClasses
so you don't need to specify a task when running the scripts.

The -Pclose option closes the browsers so is optional.

There is a multi-tier class hierarchy to support generated Pages and custom overrides as:

Page <-- MomentumPage (framework extentions) <-- Momentum<pageName>Page (Generated pages) <-- <pageName>Page (Pages used in tests and available for customisation)

The Momentum<pageName>Page would be generated by default each time as these contain the url and the at checker with hard-coded title.
The custom <pageName>Page should be generated the first time and then only optionally subsequently otherwise any customisations would be overridden. To force
a re-generation of custom Pages use the -PoverrideCustomClasses option. These will only generate custom classes that were generated but not customised.
To force override of already customised classes use the reInitialiseCustomisedClasses option.

	gradle dN -PgeneratePages -PoverrideCustomClasses -Pclose
	
To regenerate just the generated tier

	gradle dN -PgeneratePages -Pclose

To regenerate everything

	gradle dN -PgeneratePages -PoverrideCustomClasses -PreInitialiseCustomisedClasses -Pclose
	
Note: Some customised classes have been checked in as examples
		
Then refresh eclipse or check the directories src/main/groovy for all the page objects


Step 2
------
Run the Spock tests.
As we are using the Page Object Model (POM but not Maven POM as we build with gradle), we need to compile our Page and Module classes first.
This will be done as part of the test run itself.

	gradle test

To view the Spock test reports look in the build/reports directory as normal.

Step 3
------
The following command will launch the scripted tests (which also depend on the generated tests) with firefox browser only and no task that 
does anything of importance (i.e. doNothing):

    gradle dN -PwebTests
    
To run an individual script use the testName parameter

	gradle dN -PwebTests -PtestName=<testname>
	
If you are customising the generated classes then you will need to compile them before running the scripts. As script testing is just an included file
it will run before any tasks, so we we must run gradle first in this instance. For the Spock tests it will be a single invocation.

	gradle testClasses; gradle dN -PwebTests -PtestName=<testname>

----------------------------------------------------------------------------------------

As we now using configuration via GebConfig the baseUrl is set to http://localhost:8080, but you can override this with any environment 
by adding the environment and its URL to GebConfig, e.g.

	environments {
		uat {
			baseUrl = "http://live.uat.myhosted.momentum.com"
		}
	}
The above is just an example - you need to substitute with a real URL.
Also if Basic Authentication is required you may need to install a forefox plugin such as AutoAuth to avoid having to sign in for each test.
	
Then run with the specific environment to start with that environment's base Url:

	gradle dN -PwebTests -Pgeb.env=uat 
	
Next there is a very simple configuration integration into the GebConfig to move the hardcoded titles and content in files into configuration,
which can be autogenerated together with the classes themselves at this point (TODO)

	momentum {
		siteTitlePrefix = "Spindrift Momentum -- "
		homePageTitleSuffix = 'Welcome to the home of online shopping.'
		homePageTitle = "${siteTitlePrefix}${homePageTitleSuffix}"
		registrationPageTitle = "${siteTitlePrefix}Register"
		loginPageTitle = "${siteTitlePrefix}Login"
	}
	
Then its simply accessed in the page class as:

	static at = { title == momentum.homePageTitle }

There is a MomentumPage which now sub-classes the geb.Page and is extended by the site pages.
This applies the getMomentum() method for the titles if required.

This is our first line of actual framework extension code and is only a utility method to make the configured title access easier to write.

So for example to see this in action, force the home page at checker to fail:
Page <-- MomentumPage <-- MomentumHomePage <-- HomePage
The MomentumHomePage has been generated with a hard-coded title derived from the actual site
The HomePage is the generated class for customisation, so add the following to the HomePage

	static at = { title == momentum.homePageTitle }
	
Then modify GebConfig and change either homePageTitle or homePageTitleSuffix to an invalid title.
Run the test to see it fail:

	gradle testClasses; gradle dN -PwebTests -PtestName=homePage

----------------------------------------------------------------------------------------


	
	