# Basics II - Capabilities and ParameterBinding
In this chapter you will learn how to find the right Cell depending on a product property.

The manufaturer realised that only one ColorizingCell, in which the paint always has to be changend, isn't really efficient. He decided to add another one. Now for each color there is one cell.

In order to define, which cell uses which color, first add a property `Color` to the ColorizingCell and set the color in the [Capabilities](https://github.com/PHOENIXCONTACT/MORYX-Framework/blob/dev/docs/articles/Processing/Capabilities.md) to let Moryx know what the cell is capable of. In this case using a specific color. Remember EntrySerialize means the property can be set in the ui by editing the corresponding resource.

```cs
[ResourceRegistration] 
public class ColorizingCell : Cell
{
    [DataMember]
    private PencilColor _color;

    [EntrySerialize]
    public PencilColor Color
    {
        get => _color;
        set
        {
            _color = value;
            Capabilities = new ColorizingCapabilities { Color = value };
        }
    }

    protected override void OnInitialize()
    {
        base.OnInitialize();

        Capabilities = new Colorizing​Capabilities { Color = _color };

        ...
    }

    ...
}
```

For this to work add the property `Color` to the `ColorizingCapabilities` and check if the colors of the provided Capabilities match the ones you need.

```cs
public class ColorizingCapabilities : CapabilitiesBase
{
    public PencilColor Color { get; set; }

    protected override bool ProvidedBy(ICapabilities provided)
    {
        var providedCapabilities​ = provided as Colorizing​Capabilities;
        if (providedCapabilities​ != null && providedCapabilities​.Color == Color) 
            return true;

        return false;
    }
}
```

Now MORYX knows, which cell uses which color, but it still doesn't know which color the current activity needs. This information can only be found in the product.

In order to let the activity know, which color it needs, you wil use Parameters.

Parameters are the connection between *Tasks* and *Activities*. The task, which
is equivalent to the workplan step, can be parameterized and parameters can
be transferred to the activity.
Under the hood, however, the parameters can do even more: Instances of the 
process and product will be provided to the Populate method and can thus 
get lots of information. For example, order numbers can be read from the recipe or the ProductType can be read from the process. 

Now add the color as property to the `ColorizingParameters` found in the folder `Activities`. Since the property is automatically set using the product information, it doesn't need to be displayed in the UI. 

When populating parameters, the current object always represents the parameters in the workplan, while `instance` (the parameter of the populate method) holds the parameters passed by the activity. The resource gets them during the `OnActivityStarted` method. 
Now you have to set the color of `instance` to the color of the product. In order to get the ProductType, cast the process to a `ProductionProcess` and get the ProductType from the ProductInstance.

```cs
public class ColorizingParameters : VisiualInstructionParameters
{
    public PencilColor Color { get; set; }

    protected override void Populate(IProcess process, Parameters instance)
    {
        base.Populate(process,instance);

        var parameters = (Colorizing​Parameters) instance;
        var productionProcess = (ProductionProcess) process;

        var product = (GraphitePencilType)productionProcess.ProductInstance.Type;
        parameters.Color = product.Color;
    }
}
```

The color in the `Colorizing​Parameters` defines which color is needed for the activity.
This information still has to be passed on to ProcessEngine, which is done, by adjusting the RequiredCapabilities. 
The ProcessEngine routes a product to a Cell, which can perform the next activity to be done.

Go to the `ColorizingActivity` also found in the folder `Activities` and set the color in the RequiredCapabilities. 

```cs
[ActivityResults(typeof(Colorizing​ActivityResults))]
public class Colorizing​Activity : Activity<Colorizing​Parameters>
{
    ...

    public override ICapabilities RequiredCapabilities => new Colorizing​Capabilities() { Color = Parameters.Color };

    ...
}
```

Now you have done everything to have separate Cells for each pencil color. Start your project and create two different ColorizingCells, one for each color. Do not forget to configure a driver for each cell just like shown at the end of [chapter-2](chapter-2-drivers.md).

In the first chapter you set values in your parameters using the workplans UI. In this chapter the color was automatically fetched from the product. 
This concept of not having to set the value of parameters explicitly using the UI, but instead automatically fetching them from somewhere is called `ParameterBinding`.

## When to use ParameterBinding and when write new Capabilities
In order to produce different colored pencils, you added the property `Color` to the `ColorizingCapabilities` and the `ColorizingActivityParameters`. In this paragraph you will learn different approaches and in which scenarios they should be used.

In this example both ColorizingCells are identical. The only difference is the color they are setup with.

One approach would be to create a production step for each color. Then you have a `ColorizingGreenActivity` and a `ColorizingBrownActivity`, each with matching Capabilities. 
This has the downside of you needing a separate workplan for each color, which would create a lot of boilerplate. 
Different steps should be used, when you use them in the same workplan. 
Imagine the right half of your pencil is green and the other one brown, but you only have one `ColorizingActivity`. 
In order to know which color is still missing on the pencil, you would need to know, if the pencil was already partly colored. 
Just the information of the product wouldn't be sufficient anymore. 

Instead of creating different steps, you could also create different capabilities. You could have made the `ColorizingCapabilities` abstract and then dervive `ColorizingGreenCapabilities` and `ColorizingBrownCapabilities` from it.
This should be used, when it's not possible to setup a resource in order to change its characteristic. 
Let's look at the example of printing a label. The label should be printed in a specific color and with a specific method (pad printing or laser printing). 
Nearly all printers can print any color. In worst case you have to change the color manually, but it is still possible. Changing the printing method is most of the time physically impossible. 
In that case you will create abstract `PrintingCapabilities`, which contain the color.

```cs
public abstract class PrintingCapabilities : CapabilitiesBase
{
    public Color Color { get; set; }

    protected override bool ProvidedBy(ICapabilities provided)
    {
        var providedCapabilities​ = provided as PrintingCapabilities;
        if (providedCapabilities​ != null && providedCapabilities​.Color == Color)
            return true;

        return false;
    }
}
```

Now you will derive one capability for each printing method you have.

```cs 
public class LaserPrintingCapabilities : PrintingCapabilities
{
    protected override bool ProvidedBy(ICapabilities provided)
    {
        var providedColorizing​ = provided as LaserPrintingCapabilities;
        if (providedColorizing​ == null)
            return false;

        return base.ProvidedBy(provided);
    }
}
```

In this way, it's not possible to change the printing method using the UI.

