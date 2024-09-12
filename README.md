# IMDb-graph
Developed a graph based on data retrieved from IMDb, where the vertices represent actors, and an edge connects two actors if they have appeared in the same film. The project also involved creating a ranking of actors using various centrality measures to evaluate their prominence within the network.


#include <iostream>
#include <vector>
#include <stdio.h>
#include <string.h>
#include <sstream>
#include <string>
#include <fstream>
#include <typeinfo>
#include <list>

using namespace std;

constexpr auto MAXN = 15000000;          // massimo numero di attori
constexpr auto SUBG = 1000000;           // grandezza del sottografo maggiore
vector<int> adult_film;                  // il vector dei film per adulti
vector<int> Actor_film[MAXN];            // array di MAXN vector che per ogni film indica gli attori che ne hanno preso parte
vector<int> Film_actor[MAXN];            // array di MAXN vector che per ogni attore indica i film in cui ha partecipato
int is_for_adult[MAXN];                  // array booleano che indica se un film sia per adulti o meno
string name_actor[MAXN];                 // array che associa al codice di ogni attore il suo nome
vector<int> adj[MAXN];                   // lista di adiacenza
int degree_centrality[2][MAXN];          // come primo indice inserisco act_code e come secondo il grado di adj
int degree_centrality_sub_graph[2][SUBG];// come primo indice inserisco inv_conv[act_code] e come secondo il grado di sub_adj
int ranking[2][100];                     // classifica secondo le varie misure
bool visited[MAXN];                      // indica se un nodo è stato visitato o meno durante le visite al grafo
vector<int> sub_adj[SUBG];               // lista di adiacenza del sottografo connesso più grande
int conv[SUBG];                          // converte gli indici da quelli di sub_adj a quelli di adj
int inv_conv[MAXN];                      // converte gli indici da quelli di adj a quelli di sub_adj
int sum_of_distances[SUBG];              // alla cella j salvo la somma delle distanze dal nodo preso random
int closeness_centrality[2][SUBG];       // come primo indice inserisco inv_conv[act_code] e come secondo closeness_random di sub_adj



//* BLOCCO FUNZIONI DI APPOGGIO *//
void printVector(vector<int> v) // funzione che mi stampa i vector
{
  auto n = v.size();
  if (n > 0)
  {
    cout << "[";
    if (n == 1)
      cout << name_actor[v[0]];
    else
    {
      for (auto i = 0; i < n; i++)
      {
        if (i == n - 1)
          cout << name_actor[v[i]];
        else
          cout << name_actor[v[i]] << ",";
      }
    }
    cout << "]" << endl;
  }
  else
    cout << "[]" << endl;
  return;
}


void conv_printVector(vector<int> v) // funzione che stampa i vector trasformando gli indici da quelli di sub_adj a quelli di adj
{
  auto n = v.size();
  if (n > 0)
  {
    cout << "[";
    if (n == 1)
      cout << name_actor[conv[v[0]]];
    else
    {
      for (auto i = 0; i < n; i++)
      {
        if (i == n - 1)
          cout << name_actor[conv[v[i]]];
        else
          cout << name_actor[conv[v[i]]] << ",";
      }
    }
    cout << "]" << endl;
  }
  else
    cout << "[]" << endl;
  return;
}

//* BLOCCO FUNZIONI PER CREARE IL GRAFO E PER VEDERE IL NOME DEGLI ATTORI *//
void give_me_Film_actor()  // funzione che crea Film_actor
{

  string line;

  ifstream myfile("name.basics.tsv");

  if (myfile.is_open())
  {

    while (getline(myfile, line))
    {

      int k, h, j = 1;
      int code;
      string act = line.substr(2, 7);   // il codice dell'attore senza "nm"
      int act_code = atoi(act.c_str()); // intero che mi identifica il codice dell'attore
      string tab = "\t", nline = "\n", comma = ",";
      string name;
      k = line.find(tab);
      string aux1 = line, aux2;
      aux2 = aux1.substr(k + 1, line.length());
      aux1 = aux2;
      h = aux2.find(tab);
      name = aux2.substr(0, h);

      for (auto i = 0; i < 3; i++)
      {
        k = aux1.find(tab);
        aux2 = aux1.substr(k + 1, line.length());
        aux1 = aux2;
      }

      h = aux2.find(tab);
      string profession;
      profession = aux2.substr(0, h); // qui ci sono le professioni
      string films;

      if (profession.find("actor") != string::npos || profession.find("actress") != string::npos)
      { // controllo che effettivamente sia un attore
        k = aux1.find(tab);
        aux2 = aux1.substr(k + 1, line.length());
        h = aux2.find(nline);
        films = aux2.substr(0, h);

        while (films.length() >= 9)
        { // un codice per un film è fatto da tt + 7 cifre

          string film;
          int film_code = 0;

          if (films.find(comma) != string::npos)
          { // passo induttivo: se c'è più di un film
            k = films.find(comma);
            film = films.substr(2, k - 2);
            film_code = atoi(film.c_str());
            Film_actor[act_code].push_back(film_code);
            films = films.substr(k + 1, h);
          }
          else
          { // passo base: se c'è solo un film
            film = films.substr(2, h);
            film_code = atoi(film.c_str());
            Film_actor[act_code].push_back(film_code);
            films = "";
          }
        }
      }
    }

    myfile.close();
  }

  return;
}


void give_me_Actor_film()  // funzione che crea Actor_film
{
  string line;

  ifstream myfile("title.principals.tsv");

  if (myfile.is_open())
  {

    while (getline(myfile, line))
    {
      if(line.substr(0,6) != "tconst")
      {
       int film_code;
       string string_film_code = line.substr(2,7);
       film_code = atoi(string_film_code.c_str());

        int k,h;
        string tab = "\t";
        string code_actor_string;
        string aux1 = line, aux2;

        for (auto i = 0; i < 2; i++)                     // levo questo campo intermedio che non mi interessa
        {
          k = aux1.find(tab);
          aux2 = aux1.substr(k + 1, line.length());
          aux1 = aux2;
        }

        code_actor_string = aux1.substr(2,7);            // salvo il codice di questa persona, ma non so ancora se è un attore 
        int code_actor = atoi(code_actor_string.c_str());// questo è il codice che userò se questa persona sarà un attore

        string is_actor;
        k = line.find(tab);
        aux2 = aux1.substr(k + 1, line.length());
        aux1 = aux2;
        h = aux2.find(tab);
        is_actor = aux2.substr(0, h);                    // professione della persona

        if(is_actor == "actor" || is_actor == "actress")
        {
          Actor_film[film_code].push_back(code_actor);   // aggiungo in coda il codice dell'attore che ha partecipato al vector rappresentato da un particolare film
        }
       }
    }

    myfile.close();
  }
  
  return;
}


void give_me_is_for_adult()  // funzione che crea is_for_addult 
{

  for(auto k = 0; k < MAXN; k++)
  {
    is_for_adult[k] = 0;
  }

  string line;

  ifstream myfile("title.basics.tsv");

  if (myfile.is_open())
  {
    while (getline(myfile, line))
    {

      int k, h, film_code;
      string aux1 = line, aux2, film;
      string tab = "\t";
      k = line.find(tab);
      h = line.length();
      film = line.substr(2, k - 2);
      film_code = atoi(film.c_str());
      aux1 = line.substr(k + 1, h);
      aux2 = aux1;
      
      for (auto i = 0; i < 3; i++)
      {
        k = aux1.find(tab);
        aux2 = aux1.substr(k + 1, h);
        aux1 = aux2;
      }

      string adult_code;
      k = aux1.find(tab);
      adult_code = aux1.substr(0, k);

      if(adult_code == "1")  is_for_adult[film_code] = 1;    
      
    }
    myfile.close();
  }

  return;
}


void give_me_adj()  // funzione che crea la lista di adiacenza
{
  for(auto c = 0; c < MAXN; c++)     // scorro tutto Film_actor
  {

    int adj_actor;                   // il codice dell'attore che andrò ad aggiungere volta per volta nel vector della lista di adiacenza
    auto n = Film_actor[c].size();   // numero di film a cui un attore ha partecipato
    for(auto i = 0; i < n; i++)
    {
      if(is_for_adult[Film_actor[c][i]] == 0)
      {
        auto m = Actor_film[Film_actor[c][i]].size(); // lunghezza del vector associato agli attori che hanno partecipato ad un film

        for(auto j = 0; j < m; j++)  // scorro il vector associato agli attori che hanno partecipato ad un film
        {
          // non inserisco l'attore nella propria lista di adiacenza
          if(c != Actor_film[Film_actor[c][i]][j])  
          {
            auto l = adj[c].size();
            auto k = 0;
            // passo base: all'inizio aggiungo un elemento
            if(l == 0)  adj[c].push_back(Actor_film[Film_actor[c][i]][j]);
            //passo induttivo: controllo che nel vector non ci sia un attore già inserito in precedenza
            else
            {
              while((k < l) && (adj[c][k] != Actor_film[Film_actor[c][i]][j]))
              {
                // questo if inserisce l'elemento solo dopo aver scandito tutto il vector, cioè a lunghezza l-1
                if(k == l-1)  adj[c].push_back(Actor_film[Film_actor[c][i]][j]);
                k++;
              }
            }
          }
        }
      }
    }
  }
 
  return;
}


void give_me_name_actor()  // funzione che crea name_actor 
{
  string line;

  ifstream myfile("name.basics.tsv");

  if (myfile.is_open())
  {

    while (getline(myfile, line))
    {

      int k, h, j = 1;
      int code;
      string act = line.substr(2, 7);   // il codice dell'attore senza "nm"
      int act_code = atoi(act.c_str()); // intero che mi identifica il codice dell'attore
      string tab = "\t", nline = "\n";
      string name;
      k = line.find(tab);
      string aux1 = line, aux2;
      aux2 = aux1.substr(k + 1, line.length());
      aux1 = aux2;
      h = aux2.find(tab);
      name = aux2.substr(0, h);         // il nome dell'attore

      for (auto i = 0; i < 3; i++)
      {
        k = aux1.find(tab);
        aux2 = aux1.substr(k + 1, line.length());
        aux1 = aux2;
      }

      h = aux2.find(tab);
      string profession;
      profession = aux2.substr(0, h);  // qui ci sono le professioni
      string films;

      if (profession.find("actor") != string::npos || profession.find("actress") != string::npos)
      { // controllo che effettivamente sia un attore
        k = aux1.find(tab);
        aux2 = aux1.substr(k + 1, line.length());
        h = aux2.find(nline);
        films = aux2.substr(0, h);
        name_actor[act_code] = name;

      }
    }

    myfile.close();
  }
  return;
}


//* BLOCCO CREAZIONE DELLE MISURE DI CENTRALITÀ SU ADJ*//
void move(int arr[100], int pos, int x)
{
  for(auto i = pos; i < 99; i++)
  {
    int aux = arr[i];
    arr[i] = x;    
    x = aux;
  }
  arr[99] = x; 

  return;
}


void first_100_rank(int c, int sx, int dx, int arr[2][MAXN])
{
  while(sx < dx)
  {
    if(arr[1][c] > ranking[1][(sx+dx)/2])  dx = (sx+dx)/2;
    
    else  sx = (sx+dx)/2 +1;
  }
  
  move(ranking[1], sx, arr[1][c]);
  move(ranking[0], sx, arr[0][c]);

  return;
}


void give_me_degree_rank()  // funzione che crea degree_rank
{
  // riempio degree_centrality con i dati giusti, cioè nella prima cella inserisco act_code e nella seconda il grado
  for(auto c = 0; c < MAXN; c++)
  {
    auto s = adj[c].size();
    degree_centrality[0][c] = c;
    degree_centrality[1][c] = s;
  }

  // inserisco uno alla volta gli attori nel ranking dei più influenti
  for(auto c = 1; c < MAXN; c++)
  {
    if(degree_centrality[1][c] > ranking[1][99])  first_100_rank(c, 0, 99, degree_centrality);
  }

  return;
}


//* COSTRUISCO LA LISTA DI ADIACENZA DEL SOTTOGRAFO CONNESSO PIÙ GRANDE PER UTILIZZARE MEGLIO LE MISURE DI CENTRALITÀ *//
void iterative_DFS(int v)
{
  for (auto u=0; u < MAXN; u++)
  {
    visited[u] = false;
  }

  vector<int> stack;                    // utilizzo il vector come se fosse uno stack per implementare DFS iterativo
  stack.push_back(v);                   // inserisco il nodo
  int i = 0;
 
  while (!stack.empty())                // continuo finché non è vuoto, cioè finché non ho visitato tutta la componente connessa
  {
    int s = stack.size();
    v = stack[s-1];
    stack.pop_back();

    if(visited[v] == false)             // se non ho visitato il nodo leggo la lista di adiacenza e leggo tutti i nodi presenti
    {
      visited[v] = true;
      conv[i] = v;
      inv_conv[v] = i;
      i++;
      s = adj[v].size();
      for (auto j = 0; j < s; j++)
      {
        int u = adj[v][j];

        if (visited[u] == false) stack.push_back(u);
        else if(visited[u] == true)      // se ho visitato il nodo allora inserisco gli elementi nelle rispettive liste di adiacenza
        {
          sub_adj[inv_conv[u]].push_back(inv_conv[v]);
          sub_adj[inv_conv[v]].push_back(inv_conv[u]);
        }
      }
    }
  }
  return;
}


//* CALCOLO IL GRADO SUL SOTTOGRAFO *//
void first_100_rank_sub_graph(int c, int sx, int dx, int arr[2][SUBG])
{
  while(sx < dx)
  {
    if(arr[1][c] > ranking[1][(sx+dx)/2])  dx = (sx+dx)/2;
    
    else  sx = (sx+dx)/2 +1;
  }
  
  move(ranking[1], sx, arr[1][c]);
  move(ranking[0], sx, arr[0][c]);

  return;
}


void give_me_degree_rank_sub_graph()  // funzione che crea degree_rank di sub_adj
{
  // riempio degree_centrality con i dati giusti, cioè nella prima cella inserisco act_code e nella seconda il grado
  for(auto c = 0; c < SUBG; c++)
  {
    auto s = sub_adj[c].size();
    degree_centrality_sub_graph[0][c] = c;
    degree_centrality_sub_graph[1][c] = s;
  }

  // inserisco uno alla volta gli attori nel ranking dei più influenti
  for(auto c = 1; c < SUBG; c++)
  {
    if(degree_centrality_sub_graph[1][c] > ranking[1][99])  first_100_rank_sub_graph(c, 0, 99, degree_centrality_sub_graph);
  }

  return;
}


//* BLOCCO CREAZIONE RANDOM CLOSENESS CENTRALITY SUL SOTTOGRAFO*//
void give_me_sum_of_distances(int k)
{
  int distance = 0;

  for(auto i = 0; i < MAXN; i++)
  {
    visited[i] = false;
  }
  
  list<int> back;               // implementata come std::list del C++: testa si chiama front e coda si chiama back
  back.push_back(k);            // il nodo u di partenza è l'unico nella coda
  visited[k] = true;            // (in)variante BFS: nella coda solo nodi visitati, la cui distanza da u è nota
  
  while( !back.empty() ){
    // estrazione della testa della coda 
    auto u = back.front();   // FIFO: primo della coda (non viene estratto) e la sua distanza è definita
    back.pop_front();        // FIFO: estrazione effettuata
    // ogni vicino v di u va in coda solo se non è ancora visitato 
    for ( auto v: adj[u] )
    {
      if (!visited[v])
      {
        distance++;
        sum_of_distances[inv_conv[v]] = sum_of_distances[inv_conv[v]] + distance;
        back.push_back(v);
        visited[v] = true;
      }
    }
  }
  
  return;
}


int give_me_dimension_subgraph(int v)
{
  int dimension_subgraph = 1;
  
  for (auto u=0; u < MAXN; u++)
  {
    visited[u] = false;
  }

  vector<int> stack;                    // utilizzo il vector come se fosse uno stack per implementare DFS iterativo
  stack.push_back(v);                   // inserisco il nodo
 
  while (!stack.empty())                // continuo finché non è vuoto, cioè finché non ho visitato tutta la componente connessa
  {
    int s = stack.size();
    v = stack[s-1];
    stack.pop_back();

    if(visited[v] == false)             // se non ho visitato il nodo leggo la lista di adiacenza e leggo tutti i nodi presenti
    {
      visited[v] = true;
      dimension_subgraph++;

      s = adj[v].size();
      for (auto j = 0; j < s-1; j++)
      {
        int u = adj[v][j];

        if (visited[u] == false) stack.push_back(u);
      }
    }
  }
 
  return dimension_subgraph;
}


void give_me_random_Closeness()             // funzione che mi costruisce random_closeness_rank
{

  int k = give_me_dimension_subgraph(93);   // calcolo la dimensione del sottografo che mi interessa
  int n = 20;

  srand(time(0));
  for(auto c = 0; c < n; c++)
  {
    int r = rand()%k;                       // genero i valori random 
    give_me_sum_of_distances(r);            // calcolo le distanze dei nodi dai valori random
  }

  for(auto j = 0; j < k; j++)
  {
    int c = ((k-1)*n)/(k*(sum_of_distances[j]));
    closeness_centrality[0][c] = j;
    closeness_centrality[1][c] = c;
  }

  for(auto c = 1; c < SUBG; c++)
  {
    if(closeness_centrality[1][c] > ranking[1][99])  first_100_rank_sub_graph(c, 0, 99, closeness_centrality);
  }

  return;
}


int main()
{
/* chiamo le funzioni per creare la lista di adiacenza*/
  give_me_Film_actor();
  give_me_is_for_adult();
  give_me_Actor_film();
  give_me_adj();
  give_me_name_actor();
  
  // calcolo la classifica e stampo i primi 100 secondo la misura del grado di adj e di "Closeness"
  iterative_DFS(93);

  give_me_random_Closeness();

  for(auto j = 0; j < 100; j++)
  {
    cout << name_actor[conv[ranking[0][j]]] << endl;
    cout << ranking[1][j] << endl;
  }

  give_me_degree_rank_sub_graph();
  
  for(auto j = 0; j < 100; j++)
  {
    cout << name_actor[conv[ranking[0][j]]] << endl;
    cout << ranking[1][j] << endl;
  }


  return 0;
}


