---
layout: post
title: 'Paper vs. Steel #1'
date: '2012-12-12T17:40:00.000Z'
author: Richard Bradshaw
categories:
- Flaky Automation
- Automation in Testing
- Selenium WebDriver
- GUI Automation
- Automated Checking
- Automation Design
tags:
- Flaky Automation
- Selenium WebDriver
modified_time: '2012-12-12T17:40:27.428Z'
blogger_id: tag:blogger.com,1999:blog-8318661666872903125.post-7629458593387763548
blogger_orig_url: http://www.thefriendlytester.co.uk/2012/12/PaperVsSteel1.html
permalink: /2012/12/PaperVsSteel1.html
comments: true
---

I want to share my approach to making your page object methods a bit more "less flaky" but also easier to maintain and more flexible. Am not going to try and cover everything in one post, so here is the first.

### Populating a field on a page.  
{% highlight csharp %}
public void PopulateUsernameField(string username)
{
    WebDriver.FindElement(By.Id("username")).SendKeys(username);
}
{% endhighlight %}

Excellent, we all know how to do this. So if we run this, unless we get an exception, we just expect the username field is now populated. Well lets change it to a bool so our tests have something to assert against.

{% highlight csharp %}
public bool PopulateUsernameField(string username)   
{  
  WebDriver.FindElement(By.Id("username")).SendKeys(username);  
  return true;  
}
{% endhighlight %}

So now, we can assert this method returns true, and just let an exception inform us that it has failed. Still not great, we are either going to get true or exception, so lets add a try/catch so we can catch exceptions and return false.

{% highlight csharp %}
public bool PopulateUsernameField(string username)
{  
  try  
  {  
    WebDriver.FindElement(By.Id("username")).SendKeys(username);  
  }  
  catch (Exception)  
  {  
    return false;  
  }  
  return true;  
}
{% endhighlight %}

So now we would be able to use this method for positive and negative testing, if we were expecting an exception, our test would be asserting a false response. 

But what about the different exceptions we could receive, if we are always returning false, we don't know what caused it, so for a start we could output this to the console, this introduces a manual task, however I will post about more complex approaches in different post.

{% highlight csharp %}
public bool PopulateUsernameField(string username)  
{  
  try  
  {  
    WebDriver.FindElement(By.Id("username")).SendKeys(username);  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(ex.Message);  
    return false;  
    }  
    return true;  
}
{% endhighlight %}

So making it a bit more robust, we know we have to interact with an element, so why don't we wait for that element first, if its there, we carry on if not lets wait a bit.

{% highlight csharp %}
public bool PopulateUsernameField(string username, int usernameTimeout)  
{  
  WebDriverWait waitForUsernameField = new WebDriverWait(WebDriver, TimeSpan.FromSeconds(usernameTimeout));  
  waitForUsernameField.Message = string.Format("Timed out after waiting {0} seconds for the username field using {1} identifier", usernameTimeout, _locators["txtUsername"]);  
  waitForUsernameField.Until(d => d.FindElement(_locators["txtUsername"]));  
  try  
  {  
    WebDriver.FindElement(By.Id("username")).SendKeys(username);  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(ex.Message);  
    return false;  
  }  
  return true;  
}
{% endhighlight %}

So we have introduced a few new things in this code snippet, firstly I have included my locators dictionary, this approach can be found <a href="http://www.thefriendlytester.co.uk/2012/11/MyPageObjectMach1.html">here</a>. Secondly, I have included a WebDriverWait object, this can be found in the "OpenQA.Selenium.Support.UI" name space.  
I have added the time to wait as a parameter to the method, I do this as sometimes you might be testing the speed of the application, other times you may want to use the global timeout, this gives you that flexibility. Also added an error message for when this wait expires, providing specific error to reduce investigation time upon a failure.

So we now wait, and can catch if there is a problem, but we can do more. Lets check that the element is enabled, you might say, why wouldn't it be, who knows, but by doing this we can provide a specific message again reducing investigation time.

{% highlight csharp %}
public bool PopulateUsernameField(string username, int usernameTimeout)  
{  
WebDriverWait waitForUsernameField = new WebDriverWait(WebDriver, TimeSpan.FromSeconds(usernameTimeout));  
waitForUsernameField.Message = string.Format("Timed out after waiting {0} seconds for the username field using {1} identifier", usernameTimeout, _locators["txtUsername"]);  
waitForUsernameField.Until(d => d.FindElement(_locators["txtUsername"]));  
if (WebDriver.FindElement(_locators["txtUsername"]).Enabled == false)  
{  
  Console.WriteLine("The username field was not enabled");  
  return false;  
}  
try  
{   WebDriver.FindElement(By.Id("username")).SendKeys(username);        }  
catch (Exception ex)  
{  
  Console.WriteLine(ex.Message);  
  return false;  
}  
return true;  
}
{% endhighlight %}

Ok, so we are getting there, but how do we know the field is now populated with what we asked WebDriver to populate it with, we don't, so lets check that as well.

{% highlight csharp %}
public bool PopulateUsernameField(string username, int usernameTimeout)  
{  
  WebDriverWait waitForUsernameField = new WebDriverWait(WebDriver, TimeSpan.FromSeconds(usernameTimeout));  
  waitForUsernameField.Message = string.Format("Timed out after waiting {0} seconds for the username field using {1} identifier", usernameTimeout, _locators["txtUsername"]);  
  waitForUsernameField.Until(d => d.FindElement(_locators["txtUsername"]));  
  if (WebDriver.FindElement(_locators["txtUsername"]).Enabled == false)  
  {  
    Console.WriteLine("The username field was not enabled");  
    return false;  
  }  
  try  
  {  
    WebDriver.FindElement(_locators["txtUsername"]).Clear();  
    WebDriver.FindElement(_locators["txtUsername"]).SendKeys(username);  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(string.Format("Exception occured whilst trying to populate the username field. Exception: {0}", ex.Message));  
    return false;  
  }  
  string txtUsernameValue = null;  
  try  
  {  
    txtUsernameValue = WebDriver.FindElement(_locators["txtUsername"]).GetAttribute("value");  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(string.Format("Was unable to read the value attribute from the username field. Exception: {0}", ex.Message));  
    return false;  
  }  
    if (txtUsernameValue != username)  
    {  
      Console.WriteLine(string.Format("The username field doesn't contain {0}, it contains {1}", username, txtUsernameValue));  
      return false;  
    }  
    return true;  
}
{% endhighlight %}

You will see that I have also added to clear the field before populating it, not everyone will want this, but it something to consider, depending on your applications behaviour. You could have a method to clear it first.

So as you can see there are several things we do to reduce flakiness, and also reduce investigation time.

So I also said maintenance, well obviously we aren't going to write all this everytime we want to populated a field, so we extract this to a library or I have added this to my default page object as a protected method, a few tweaks and we end up with this:

{% highlight csharp %}
///<summary>  
/// A method for populating a txt field  
/// </summary>  
/// <param name="identifier">The locator used to find the text field</param>  
/// <param name="value">The value you wish to populate the field with</param>  
/// <param name="timeout">How long WebDriver should wait for the element to appear</param>  
/// <param name="friendlyRef">A friendly name for the field, used for error reporting</param>  
/// <returns>True is successfully populated</returns>  
protected bool PopulateField(By identifier, string value, int timeout, string friendlyRef)  
{  
  WebDriverWait waitForField = new WebDriverWait(WebDriver, TimeSpan.FromSeconds(timeout));  
  waitForField.Message = string.Format("Timed out after waiting {0} seconds for the {1} field using {2} identifier", timeout, friendlyRef, identifier.ToString());  
  waitForField.Until(d => d.FindElement(identifier));  
  if (WebDriver.FindElement(identifier).Enabled == false)  
  {  
    Console.WriteLine(string.Format("The {0} field was not enabled", friendlyRef));  
    return false;  
  }  
  try  
  {  
    WebDriver.FindElement(identifier).Clear();  
    WebDriver.FindElement(identifier).SendKeys(value);  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(string.Format("Exception occured whilst trying to populate the {0} field. Exception: {1}", friendlyRef, ex.Message));  
    return false;  
  }  
  string txtValue = null;  
  try  
  {  
    txtValue = WebDriver.FindElement(identifier).GetAttribute("value");  
  }  
  catch (Exception ex)  
  {  
    Console.WriteLine(string.Format("Was unable to read the value attribute from the {0} field. Exception: {1}", friendlyRef, ex.Message));
    return false;  
  }  
  if (txtValue != value)  
  {  
    Console.WriteLine(string.Format("The {0} field doesn't contain {1}, it contains {2}", friendlyRef, value, txtValue));  
    return false;  
  }  
  return true;  
}
{% endhighlight %}

We can than call this method from our page objects, as all with have access to it, as they all inherit from DefaultPage.

{% highlight csharp %}
public bool PopulateUsernameField(string username, int timeout)  
{  
  return PopulateField(_locators["txtUsername"], username, timeout, "username");  
}
{% endhighlight %}

As mentioned, there is better approach to handling exception, but hopefully you get an idea of what else you can do to reduce flakiness in some of your methods, but also reduce investigation time with failures, with smarter reporting.

I have deliberately called this post Paper vs, Steel, paper was just chosen as I was trying to thing of something weak, and there is always a pile of it on my desk, as its my favourite testing tool.

Then Steel, its a very strong material, but not the strongest, so while I think my approach is strong, I know its not the best, so if you can see a way I can improve it, then let me know, and perhaps future posts can be about titanium methods!