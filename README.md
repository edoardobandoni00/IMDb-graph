# IMDb-graph
This project provides a comprehensive C++ implementation for analysing **actor and movie networks** using graph-based techniques. It processes IMDb datasets to build a graph where the nodes represent `actors` and the edges represent their shared appearances in films. The program generates an adjacency list for **actors**, excluding adult films, and includes centrality measures such as `degree` and `closeness centrality`.

The project begins by processing IMDb datasets (`name.basics.tsv`, `title.basics.tsv`, and `title.principals.tsv`) to map **actors** to films and films to **actors**. This results in two main data structures: `Film_actor`, which stores the films associated with each **actor**, and `Actor_film`, which includes the actors participating in each film. Using these mappings, the program constructs an adjacency list (`adj`), where actors are connected based on their co-appearances in non-adult films. The graph creation also includes functionality to build a subgraph representing the largest connected component, which allows for a more focused analysis of significant actor clusters.

The project calculates degree centrality for the graph. This generates a ranking of the top 100 actors based on their influence within the network. Additionally, the program estimates closeness centrality for the subgraph using a randomised approach. These centrality measures offer insights into the prominence and connectivity of actors within the movie network.

To support the main functionalities, several helper functions are included for vector manipulation, actor name printing, and translating between actor indices in the full graph and the subgraph. This ensures efficient handling of large datasets and smooth execution of the program’s various tasks.

Overall, this project demonstrates how graph theory and centrality measures can be applied to analyse real-world datasets, particularly in the context of understanding actor relationships within the film industry.

