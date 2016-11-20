---
layout: post
title:  "Parsing HL7 Z Segments with nHapi"
date:   2016-11-5
---

<p class="intro"><span class="dropcap">R</span>ecently I was tasked with parsing a non standard Zxx HL7 segment. After combing the internet, I was able to find one article that described just what I needed. However, the information was out-dated and hard to extend.</p>

<h2> Getting Started </h2>

<p>A sample project can be found <a href="https://github.com/jackfacts/HL7SampleZSegment">here</a>. </p>


<p>To start off first we want to create a couple folders. These folders are based on a naming convention that is expected by the <a href="https://github.com/duaneedwards/nHapi/blob/master/NHapi20/NHapi.Base/Parser/DefaultModelClassFactory.cs">DefaultModelClassFactory</a> in nHapi. If you would wish to use a different schema it is easily accomplished by implmenting the <a href="https://github.com/duaneedwards/nHapi/blob/master/NHapi20/NHapi.Base/Parser/IModelClassFactory.cs">IModelClassFactory</a> and passing that into the Parser. The three folders that need to be created are: Group, Message, and Segment.</p>

![Screen shot of directories]({{ site.url }}assets/img/hl7folders.png)

<p>Now that we have created a matching structure lets create a Z Segment in the Segment directory. I start by creating a new class and naming it according to the segment that I need. In my example project that is <a href="https://github.com/jackfacts/HL7SampleZSegment/blob/master/jackfacts.HL7.V24/Segment/ZIS.cs">ZIS.cs</a>. The first thing to note when creating the class is to ensure that the class has the Serializable attribute. If you happen to get serialization exceptions this is the likey culprit. While it is not neccesary to inherit from Zxx I did so for clarity's sake. However, it is neccessary for the class to inherit from <a href="https://github.com/duaneedwards/nHapi/blob/master/NHapi20/NHapi.Base/Model/AbstractSegment.cs">AbstractSegment</a> </p>

{% highlight C# %}

    [Serializable]
    public sealed class ZIS : Zxx
    {
        private const string DefaultErrorMessage = "Unexpected problem obtaining field value.  This is a bug.";

        private const string DefaultExceptionMessage = "An unexpected error ocurred";

        public ZIS(IGroup parent, IModelClassFactory factory) : base(parent, factory)
        {
            IMessage message = Message;

            try
            {
                add(typeof(CE),
                    true,
                    1,
                    250,
                    new object[]
                    {
                        message
                    },
                    "Idetifier");
            }
            catch (HL7Exception he)
            {
                HapiLogFactory.GetHapiLog(GetType())
                    .Error("Can't instantiate " + GetType()
                               .Name,
                        he);
            }
        }

        public CE Identifier
        {
            get
            {
                try
                {
                    return (CE) GetField(1, 0);
                }
                catch (HL7Exception he)
                {
                    HapiLogFactory.GetHapiLog(GetType())
                        .Error(DefaultErrorMessage, he);
                    throw new Exception(DefaultExceptionMessage, he);
                }
                catch (Exception ex)
                {
                    HapiLogFactory.GetHapiLog(GetType())
                        .Error(DefaultErrorMessage, ex);
                    throw new Exception(DefaultExceptionMessage, ex);
                }
            }
        }

    }

{% endhighlight %}

<p>The properties on ZIS are setup in a very similar way as nHapi does, with just a public getter and its intialization done by calling the add method in the constructor. Now that we have the Z Segment, it can now be used directly on a message or through a Group. In the example project I created a <a href="https://github.com/jackfacts/HL7SampleZSegment/tree/master/jackfacts.HL7.V24/Group">ZIS_GROUP</a>, which allows the segment to be an array of segments on a message. If the Z segment was needed directly on the message than it would just need to be added as a property directly on the message. Below is the ZIS_GROUP</p>

{% highlight C# %}

 [Serializable]
    public class ZIS_GROUP : AbstractGroup
    {
        public ZIS_GROUP(IGroup parent, IModelClassFactory factory) : base(parent, factory)
        {
            try
            {
                add(typeof(ZIS), false, false);
            }
            catch (HL7Exception e)
            {
                HapiLogFactory.GetHapiLog(GetType())
                    .Error(
                        "Unexpected error creating MFR_M01_MF_QUERY - this is probably a bug in the source code generator.",
                        e);
            }
        }

        /// <summary>
        ///     Returns ZIS (any Z segment) - creates it if necessary
        /// </summary>
        public ZIS ZIS
        {
            get
            {
                try
                {
                    return (ZIS) GetStructure(nameof(ZIS));
                }
                catch (HL7Exception e)
                {
                    HapiLogFactory.GetHapiLog(GetType())
                        .Error(
                            "Unexpected error accessing data - this is probably a bug in the source code generator.", e);
                    throw new Exception("An unexpected error ocurred", e);
                }
            }
        }
    }

{% endhighlight %}


<p>The last step is to register the new models with the <a href="https://github.com/duaneedwards/nHapi/blob/master/NHapi20/NHapi.Base/PackageManager.cs">PackageManager</a> that is called by the DefaultModelClassFactory when initializing the parser. This is done by adding the following configuration to the App.Config/Web.Config of the executing assembly.</p>


{% highlight xml %}

<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="Hl7PackageCollection" type="NHapi.Base.Model.Configuration.HL7PackageConfigurationSection, NHapi.Base" />
  </configSections>
  <Hl7PackageCollection>
    <HL7Package name="jackfacts.HL7.V24" version="2.4" />
  </Hl7PackageCollection>
</configuration>

{% endhighlight %}


<p>Here you will want to chage the HL7Package name="jackfacts.HL7.V24" and version to match those of your assembly. If the parser fails to find your assembly, it is easily debugged by overriding the GetSegmentClass method of the DefaultModelClassFactory.</p>