[![Build status](https://ci.appveyor.com/api/projects/status/cgo827o4f74lmf9w/branch/master?svg=true)](https://ci.appveyor.com/project/kevinmarquette/PSGraph/branch/master) [![Documentation Status](https://readthedocs.org/projects/psgraph/badge/?version=latest)](http://psgraph.readthedocs.io/en/latest/?badge=latest)
    

# PSGraph

PSGraph is a helper module implemented as a DSL (Domain Specific Language) for generating GraphViz graphs. The goal is to make it easier to generate graphs using Powershell. The DSL adds these commands that are explained below.

* [graph](http://psgraph.readthedocs.io/en/latest/Command-Graph/)
* [edge](http://psgraph.readthedocs.io/en/latest/Command-Edge/)
* [node](http://psgraph.readthedocs.io/en/latest/Command-Node/)
* [subgraph](http://psgraph.readthedocs.io/en/latest/Command-SubGraph/)
* [rank](http://psgraph.readthedocs.io/en/latest/Command-Rank-Advanced/)

## What is GraphViz?

[Graphviz](http://graphviz.org/) is open source graph visualization software. Graph visualization is a way of representing structural information as diagrams of abstract graphs and networks. It has important applications in networking, bioinformatics,  software engineering, database and web design, machine learning, and in visual interfaces for other technical domains. 

## Project status?
Beta release. The core module work and documentation is fleshed out. The simple features are implemented but there are other features of the DOT language that I have not used much myself. The command names and arguments of the existing commands should be stable now. As always, more testing still needs to be done.

# GraphViz and the Dot format basics
The nice thing about GraphViz is that you can define nodes and edges with a simple text file in the Dot format. The GraphViz engine handles the layout, edge routing, rendering and creates an image for you. 

Here is a sample Dot file.

    digraph g {
        "Start"->"Middle"
        "Middle"->"End"
    }

This will produce a graph with 3 nodes with the names `Start`, `Middle` and `End`. There will be an edge line connecting them in order. Go checkout the [GraphViz Gallery](http://www.graphviz.org/gallery/) to see some examples of what you can do with it.

## How PSGraph can help
we can create those Dot files with PSGraph in Powershell. Here is that same graph as above.

    digraph g {
        edge Start Middle
        edge Middle End
    }

Here is a second way to approach it now.

    $objects = @("Start","Middle","End")
    digraph g {
        edge $objects
    }

 The real value here is that I can specify a collection to process. This allows the graph to be data driven.

# PSGraph Commands
I tried to keep a syntax that was similar to GraphViz but offer the flexibility of Powershell. If you have worked with GraphViz before, then you should feel right at home.

## Graph or digraph
This is the container that holds graph elements. Every valid GraphViz graph has one. `digraph` is just an alias of `graph` so I am going to use the shorter `graph` for the rest of the readme.

    graph g {        
    }

## Edge
This defines an edge or connection between nodes. This command accepts a list of strings or an array of strings. It will connect them all in sequence. This should be placed inside a `graph` or `subgraph`. `-From` and `-To` parameters are available if you want more verbosity.

    graph g {
        edge -From first -To Second
        edge one two three four
        edge (1..5)
    }

If you supply two arrays, it will cross multiply them. So each item on the left with have a path to each item on the right.

    graph g {
        edge (1,3) (2,4)
    }
    
Because this is Powershell, we can mix in normal commands. Take this example.

    $folders = Get-ChildItem -Directory -Recurse
    graph g {
        $folders | %{edge $_.Name $_.Parent} 
    }

I also support edge attributes that are defined in the DOT specification by using a hashtable.

   graph g {
       edge "Point One" "Point Two" @{label='A line'}
   }

## Node
The Node command allows you to introduce a node on the graph before an edge is created. This is used to give specific nodes different attributes or to place them in subgraphs. The node command will also accept an array of values and a hashtable of attributes.

    graph g {
        node start @{shape='house'}
        node end @{shape='invhouse'}
        edge start,middle,end
    }

This is the exact Dot output generated those node commands.

    digraph g {

        "start" [shape="house"]
        "end" [shape="invhouse"]
        "start"->"middle" 
        "middle"->"end" 
    }

## Rank
The rank command will place the specified nodes at the same level in the chart. The layout can get a bit wild sometime and gives you some control.

    graph g {
        rank 2 4 6 8
        rank 1 3 5 7
        edge (1..8)
    }

## SubGraph
This lets you box of and cluster nodes together in the graph. These can be nested. The catch is that you need to number them starting at 0. Attributes are supported with the `-Attributes` parameter.

    graph g {
        subgraph 0 -Attributes @{label='DMZ'} {
            node Web1,Web2
        }
        edge web1 database
        edge web1 database
    }

# More complex example
We can pull that all together and generate quite the data driven driven diagram.

    $webServers = 'Web1','Web2','web3'
    $apiServers = 'api1','api2'
    $databaseServers = 'db1'

    graph site1 {
        # External/DMZ nodes
        subgraph 0 -Attributes @{label='DMZ'} {
            node 'loadbalancer' @{shape='house'}
            rank $webServers
            node $webServers @{shape='rect'}
            edge 'loadbalancer' $webServers
        }

        subgraph 1 -Attributes @{label='Internal'} {
            # Internal API servers
            rank $apiServers
            node $apiServers   
            edge $webServers -to $apiServers
        
            # Database Servers
            rank $databaseServers
            node $databaseServers @{shape='octagon'}
            edge $apiServers -to $databaseServers
        }    
    }



# Installing PSGraph
Make sure you are running Powershell 5.0 (WMF 5.0). I don't know that it is a hard requirement at the moment but I plan on using 5.0 features.

    # Install GraphViz from the Chocolatey repo
    Register-PackageSource -Name Chocolatey -ProviderName Chocolatey -Location http://chocolatey.org/api/v2/
    Find-Package graphviz | Install-Package -ForceBootstrap

    # Install PSGraph from the Powershell Gallery
    Find-Module PSGraph | Install-Module

    # Import Module
    Import-Module PSGraph

For OSX, you can use brew to install graphviz.

    brew install graphviz

# Generating a graph image
I am still working out the workflow for this, but for now just do this.

    # Create graph in variable
    $dot = graph g {
        edge hello world
    }

    # Save to file
    Set-Content -Path $env:temp\hello.vz -Value $dot
    
    Export-PSGraph -Source $env:temp\hello.vz -Destination $env:temp\hello.png -ShowGraph

The export can be done more in line if needed.

    graph g {
        edge hello world
    } | Export-PSGraph -ShowGraph