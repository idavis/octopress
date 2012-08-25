---
layout: post
title: "Prototypal Inheritance Using PowerShell (Part 3): Mixins"
date: 2012-08-25 18:10
comments: true
categories: 
published: false
---
If you didn't read [Part 1][] and [Part 2][], I would recommend reading them as I build off of their functionality and theory.

Each derivation/extension has the chance to replace override previously declared functionality.

Because we aren't dealing with classical inheritance, we can have

``` ps1
filter Mixin-Math {
  $_ | Add-Property PI ([Math]::PI) Constant
  $_ | Add-Property E ([Math]::E) Constant
  $_ | Add-Function Sqrt {param($value)[Math]::Sqrt($value)}
  $_ | Add-Function Hypotenuse {param($a,$b) $this.Sqrt($a*$a+$b*$b)}
}

filter Mixin-Circular {
  param($radius = 3.0)
  $_ | Mixin-Math
  $_ | Add-Property Radius $radius
  $_ | Add-ScriptProperty Diameter {$this.Radius * 2.0}
  $_ | Add-ScriptProperty Circumference {$this.Diameter * $this.PI}
  $_ | Add-ScriptProperty BaseArea {$this.Radius * $this.Radius * $this.PI}
}

filter Mixin-Cylindrical {
  param($radius = 3.0, $height = 4.0)
  $_ | Mixin-Circular $radius
  $_ | Add-Property Height $height
  $_ | Add-ScriptProperty LateralHeight {$this.Height}
  $_ | Add-ScriptProperty LateralArea {$this.Circumference * $this.LateralHeight}
  $_ | Add-ScriptProperty SurfaceArea {$this.BaseArea + $this.LateralArea}
  # override/replace
  $_ | Add-ScriptProperty BaseArea {2 * $this.Radius * $this.Radius * $this.PI}
}
```


``` ps1
function New-Cylinder {
  param($radius = 3.0, $height = 4.0)
  $prototype = new-prototype
  $prototype | Update-TypeName
  $prototype | Mixin-Cylindrical $radius $height
  $prototype
}

$cylinder = New-Cylinder -radius 1 -height 1
$cylinder | Get-Member -View Extended
$cylinder | fl

   TypeName: Prototype#Cylinder

Name          MemberType     Definition
----          ----------     ----------
E             NoteProperty   System.Double E=2.71828182845905
Height        NoteProperty   System.Int32 Height=1
PI            NoteProperty   System.Double PI=3.14159265358979
Radius        NoteProperty   System.Int32 Radius=1
Hypotenuse    ScriptMethod   System.Object Hypotenuse();
Sqrt          ScriptMethod   System.Object Sqrt();
BaseArea      ScriptProperty System.Object BaseArea {get=2 * $this.Radius * $this.Radius * $this.PI;}
Circumference ScriptProperty System.Object Circumference {get=$this.Diameter * $this.PI;}
Diameter      ScriptProperty System.Object Diameter {get=$this.Radius * 2.0;}
LateralArea   ScriptProperty System.Object LateralArea {get=$this.Circumference * $this.LateralHeight;}
LateralHeight ScriptProperty System.Object LateralHeight {get=$this.Height;}
SurfaceArea   ScriptProperty System.Object SurfaceArea {get=$this.BaseArea + $this.LateralArea;}


PI            : 3.14159265358979
E             : 2.71828182845905
Radius        : 1
Diameter      : 2
Circumference : 6.28318530717959
Height        : 1
LateralHeight : 1
LateralArea   : 6.28318530717959
SurfaceArea   : 12.5663706143592
BaseArea      : 6.28318530717959
```


``` ps1
function New-Cone {
  param($radius = 3.0, $height = 4.0)
  $prototype = new-prototype
  $prototype | Update-TypeName
  $prototype | Mixin-Cylindrical $radius $height
  # override/replace Calculations
  $prototype | Add-ScriptProperty BaseArea {$this.Radius * $this.Radius * $this.PI}
  $prototype | Add-ScriptProperty LateralHeight {$this.Hypotenuse($this.Radius, $this.Height)}
  $prototype | Add-ScriptProperty LateralArea {$this.PI * $this.Radius * $this.LateralHeight}
  $prototype
}


$cone = New-Cone -radius 1 -height 1
$cone | Get-Member -View Extended
$cone | fl

   TypeName: Prototype#Cone

Name          MemberType     Definition
----          ----------     ----------
E             NoteProperty   System.Double E=2.71828182845905
Height        NoteProperty   System.Int32 Height=1
PI            NoteProperty   System.Double PI=3.14159265358979
Radius        NoteProperty   System.Int32 Radius=1
Hypotenuse    ScriptMethod   System.Object Hypotenuse();
Sqrt          ScriptMethod   System.Object Sqrt();
Area          ScriptProperty System.Object Area {get=$this.LateralArea + 2.0 * $this.Pi * $this.Radius * $this.Height;}
BaseArea      ScriptProperty System.Object BaseArea {get=$this.Radius * $this.Radius * $this.PI;}
Circumference ScriptProperty System.Object Circumference {get=$this.Diameter * $this.PI;}
Diameter      ScriptProperty System.Object Diameter {get=$this.Radius * 2.0;}
LateralArea   ScriptProperty System.Object LateralArea {get=$this.Circumference * $this.LateralHeight;}
LateralHeight ScriptProperty System.Object LateralHeight {get=$this.Hypotenuse($this.Radius, $this.Height);}
SurfaceArea   ScriptProperty System.Object SurfaceArea {get=$this.BaseArea + $this.LateralArea;}




PI            : 3.14159265358979
E             : 2.71828182845905
Radius        : 1
Diameter      : 2
Circumference : 6.28318530717959
Height        : 1
LateralArea   : 8.88576587631673
SurfaceArea   : 12.0273585299065
Area          : 15.1689511834963
BaseArea      : 3.14159265358979
LateralHeight : 1.4142135623731
```


  [Part 1]: /2012/08/prototypal-inheritance-using-powershell
  [Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties