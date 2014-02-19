---
layout: post
title: "Finding and mapping Windows OLE object property values using Ruby"
date: 2012-06-23 16:37
comments: true
categories: 
tags: [ruby, windows, ole, property, values]

---

I learnt much of my Ruby doing stuff on Windows mainly because Windows was the majority platform in my day job and the platform I needed to get things done on.

In those days installing Ruby on Windows was more difficult than it is today ever since the *wonderful* folks over at [RubyInstaller](http://rubyinstaller.org/) have made it such a doddle.

Ruby on Windows has always been considered something of a second class citizen in the Ruby family.  But its really easy do some some very useful and powerful stuff: using Ruby with various Windows APIs has proved very productive for me. 

It would be remiss of me, in a first post on using Ruby with Windows, not to tip my hat at [David Mullet](http://rubyonwindows.blogspot.co.uk).  Reading David's excellent blog made me more than once expostulate "you can really do that so easily?".  Thank you David for teaching me to fish in the Ruby / Windows waters and giving me the steer on many topics.

Since then I've written quite a lot of Windows-specific code in Ruby including for Office (Excel, Outlook), WMI (the Swiss army knife of Windows APIs!) and (some) ADSI.  

One of the tasks I've found myself doing often was needing to obtain the value of an OLE object's (e.g the worksheets collection of an Excel workbook) property value and map the value (especially if it is a collection) to some more convenient Ruby form (e.g. "importing" the property into an instance of a class).

I've also wanted the mapping to be *lazy*: for example I didn't want to have to read in *all* the emails in my Outlook inbox before I could process *any* of them.  Although lazy enumeration could be done in Ruby 1.8, 1.9 brought "first class" (external) enumerators.

The code for a method to find an OLE object's property value(s) and create a mapped enumerator is almost trivial.  

<!-- more -->

(Note the below is the essence of my own method, but cut down for the example.)

``` ruby

# An example to show how to "rubify" WIN32OLE property values

require "win32ole"

def find_ole_property_value(propertyName, oleTarget, &maprBlok)
	
	oleTarget.is_a?(WIN32OLE) || raise(ArgumentError, "oleTarget >#{oleTarget.class}< is not WIN32OLE", caller)
	
	oleValue = oleTarget.__send__(propertyName) rescue nil  #  Raise exception later with useful info
	
	case 
	when oleValue.is_a?(WIN32OLE) then
		if oleValue.respond_to?(:each) then
			
			if block_given? then  # mapping wanted?
			
				# Note the collection value will only be yielded if the result of the maprBlok call is not nil
				
				eachEnum =  Enumerator.new { |g| oleValue.each.with_index { | p | r = maprBlok.call(p); r && (g.yield r) } }
				
			else
				oleValue # no mapping wanted
			end
			
		else
			block_given? ? maprBlok.call(oleValue) : oleValue  # a non-collection value can be mapped as well
		end
		
	else
		raise(ArgumentError, "oleTarget >#{oleTarget}< propertyName >#{propertyName}< oleValue >#{oleValue.class}< is not WIN32OLE", caller)
	end

end
```

Ok, so that's the basic method, below is contrived example to find the names of all the worksheets in a workbook:

``` ruby

myTestWorkbook = 'c:/temp/mytestworkbook.xlsx'  # The path to my spreadsheet

xlHandle = WIN32OLE.new('Excel.Application')  # Start an instance of Excel.  It will be invisible by default

wbHandle = xlHandle.Workbooks.Open(myTestWorkbook)  # Get the workbook OLE handle

worksheetsEnum = find_ole_property_value('Worksheets', wbHandle) do | wsHandle |  # create the enumerator
	wsHandle.Name # enumerator will yield this value i.e. the worksheet name
end

# Now do whatever with the list of worksheet names available from the enumerator

myworksheetsNames = worksheetsEnum.map { | wsName | puts("Found a worksheet called >#{wsName}<"); wsName }

wbHandle.close # be tidy

xlHandle.quit 	# be tidy
```

For completeness, this code printed the following:

``` bash
Found a worksheet called >Worksheet1<
Found a worksheet called >My Worksheet 2<
Found a worksheet called >The Third Worksheet<
```

Only scratching the surface of what's possible.