# Orchestration Woes
I hit a problem with BTS2013r2 orchestration that I don't remember coming across previously.

I had a map that required a source type of "CCPAdapter.Schemas.AdapterItem". I had intended to use the orchestration parameter "AdapterItem" for this:
![](/images/orchestration-param/2.png)
Problem was - I couldn't select the variable in the map dialog:
![](/images/orchestration-param/1.png)

So I checked and confirmed that my orchestration's input param was of multi-part message type, also called "AdapterItem", the body part of which was the correct schema:
![](/images/orchestration-param/3.png)

This had me puzzled for a while, then I noticed that the "AdapterItem" node of the Multi-part Message Types list was a faint grey colour. This is typically the case when it is being referenced from a different project. However, in this case, the multi-part type was defined in an orchestration called "CCPAdapterTypes" which was in the same project as the orchestration that was failing to compile - because I was unable to set the transform shape.

It turned out, the problem was my orchestration project had lost its reference to the schemas project - which contained the schema that the multi-part message used.

Simple error, but one for which I would have expected a more explicit notification of the cause. Hence, this blog post in case this has anyone else scratching their head - or more likely, me again in a couple of weeks once I've forgotten I even wrote this!