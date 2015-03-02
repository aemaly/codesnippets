# codesnippets
#include <queue>
#include <map>
#include <unordered_map>
#include <set>
#include <iostream>
#include <sstream>
#include <cstring>
#include <string>
#include <cstdlib>
#include <locale>
#include <vector>
#include <algorithm>
#include <random>
#include <getopt.h>
#include <climits>

using namespace std;
ostringstream os;

struct Client{
    long long timestamp;
    string name;
    bool buy; //if not buy, sell
    string equity;
    long long price; 
    long long quantity;
    long long id;

};

struct Summarize{
    long long total_commission;
    long long total_money_transferred;
    long long total_comp_trades;
    long long total_shares_traded;
};

struct Transfer{
    //string name;
    long long num_bought;
    long long num_sold;
    long long net_traded;

};

struct Getopt{

    bool median;
    bool summary;
    bool verbose;
    bool transfers; 
    bool insider; 
    bool ttt;
    set<string> insider_set;
    vector<string> ttt_set;

};


struct trav_datum{

long long time_sell;
long long time_buy;
long long amt_buy; //update buy at lowest sell
long long amt_sell; //update sell at highest buy

trav_datum() : time_sell(-1),time_buy(-1),amt_buy(-1),amt_sell(-1) {}
};


struct Time_Traveler{

    trav_datum prev;
    trav_datum curr;

};




class sell_comparator{

public:
    sell_comparator();
    bool operator()(const Client &a, const Client &b) const
    {
        if(a.price > b.price) return true;
        else if(a.price == b.price && a.id > b.id) return true;
        else return false;
    }
};

sell_comparator::sell_comparator() {}

class buy_comparator{

public:
    buy_comparator();
    bool operator()(const Client &a, const Client &b) const
    {
        if(a.price < b.price) return true;
        else if(a.price == b.price && a.id > b.id) return true;
        else return false;
    }
};

buy_comparator::buy_comparator() {}

struct Order_Book{

    priority_queue<Client, std::vector<Client>, buy_comparator> buy;
    priority_queue<Client, std::vector<Client>, sell_comparator> sell;
    priority_queue<long long, vector<long long>, greater<long long> > min_heap;
    priority_queue<long long, vector<long long>, less<long long> > max_heap;

};
    
inline string cpp_str(char* &merp)
{
    string temp = merp;
    return merp;
}

long long get_median(long long &curr_timestamp, bool inside, unordered_map<string, Order_Book> &order_book_finder, Client &client, Getopt &getopt, set<string> &sorted_keys)
{
    //FOR EACH EQUITY.....LEXICOGRAPHICALLY
    
    unordered_map<string, long long> save_medians;
    long long median_price = 0;
    //set<string> sorted_keys;
    for(auto it = order_book_finder.begin(); it !=order_book_finder.end(); ++it)
    {
        sorted_keys.insert(it->first); 
    }
    

    for(auto it = sorted_keys.begin(); it!= sorted_keys.end(); ++it)
    {
        if(order_book_finder[*it].min_heap.empty() && order_book_finder[*it].max_heap.empty())
            median_price = 0;

        else if(order_book_finder[*it].max_heap.empty())
            median_price = order_book_finder[*it].min_heap.top();

        else
        {
            if(order_book_finder[*it].min_heap.size() == order_book_finder[*it].max_heap.size())
                median_price = (order_book_finder[*it].min_heap.top() + order_book_finder[*it].max_heap.top())/2;
            else if (order_book_finder[*it].min_heap.size() > order_book_finder[*it].max_heap.size())
                median_price = order_book_finder[*it].min_heap.top();
            else
                median_price = order_book_finder[*it].max_heap.top();    

        }

        save_medians[*it] = median_price;
        if(median_price != 0 && getopt.median && inside == false) //if inside = true, don't prlong long anything, just calculate median
        {
            os<<"Median match price of "<<*it<<" at time "<<curr_timestamp<<" is $"<<median_price<<endl;
        }

    }

    return save_medians[client.equity]; //use this for insiders
}

void calculate_trade(unordered_map<string, Transfer> &transf, Summarize &final_summary, long long &curr_timestamp, long long &match_price, 
                Client &client, long long &insider_check, Getopt &getopt, unordered_map<string, Order_Book> &order_book_finder, 
                                        unordered_map<string, Time_Traveler> &timer, set<string> &sorted_keys)
{
    long long num_shares = 0;
    long long commission = 0;

    if(client.timestamp != curr_timestamp)
    { //cout<<"Client timestamp is "<<client.timestamp<<" current timestamp is" <<curr_timestamp<<endl;
        if(getopt.median)
        {
            insider_check = get_median(curr_timestamp, false, order_book_finder, client, getopt, sorted_keys);
        }
        curr_timestamp = client.timestamp;
    }

        if(client.buy && client.price != 0)
        {
            order_book_finder[client.equity].buy.push(client); //cout<<"Just pushed "<<client.name<<" long longo Buy"<<endl;
                if(getopt.ttt)    
                {
                     

                    if(timer[client.equity].curr.amt_sell != -1)
                    {
                        if(client.price > timer[client.equity].curr.amt_buy)
                        {
                            timer[client.equity].curr.amt_buy = client.price; 
                            timer[client.equity].curr.time_buy = client.timestamp;
                        }
                    }
                }//end if ttt
        }
            

        else if(!client.buy && client.price != 0)
        {
             order_book_finder[client.equity].sell.push(client); //cout<<"Just pushed "<<client.name<<" long longo Sell"<<endl;
                 if(getopt.ttt)
                 {
                     

                         if(timer[client.equity].curr.amt_sell == -1)
                         {
                             timer[client.equity].curr.amt_sell = client.price; 
                             timer[client.equity].curr.time_sell = client.timestamp;
                         }

                        else if(client.price < timer[client.equity].curr.amt_sell)
                        {
                            if((timer[client.equity].curr.amt_buy != -1 &&
                                     ((timer[client.equity].prev.amt_buy - timer[client.equity].prev.amt_sell) < 
                                         (timer[client.equity].curr.amt_buy - timer[client.equity].curr.amt_sell))) ||
                                     ((timer[client.equity].prev.amt_buy == -1)  && (timer[client.equity].prev.amt_sell == -1)))
                            {
                                timer[client.equity].prev = timer[client.equity].curr;

                                timer[client.equity].curr.amt_sell = client.price;
                                timer[client.equity].curr.time_sell = client.timestamp;
                                timer[client.equity].curr.amt_buy = -1;
                                timer[client.equity].curr.time_buy = -1;
                            }

                            else
                            {
                                timer[client.equity].curr.amt_sell = client.price;
                                timer[client.equity].curr.time_sell = client.timestamp;
                                timer[client.equity].curr.amt_buy = -1;
                                timer[client.equity].curr.time_buy = -1;
                            }
                        }
                 }
        }

        Client temp_client;

    if(order_book_finder[client.equity].buy.empty() || order_book_finder[client.equity].sell.empty())
        return;


    while(order_book_finder[client.equity].buy.top().price >= order_book_finder[client.equity].sell.top().price)
    { 
        //match price of trade is limit price of order with lower id number
        if(order_book_finder[client.equity].buy.top().id < order_book_finder[client.equity].sell.top().id)
            match_price = order_book_finder[client.equity].buy.top().price;

        else
            match_price = order_book_finder[client.equity].sell.top().price;

        if((order_book_finder[client.equity].sell.top().quantity - order_book_finder[client.equity].buy.top().quantity) >= 0)
            num_shares = order_book_finder[client.equity].buy.top().quantity; //fix this case by case

        else
            num_shares = order_book_finder[client.equity].sell.top().quantity;

        if(getopt.verbose)
        {
            os<<order_book_finder[client.equity].buy.top().name<<" purchased "<<num_shares<<" shares of "<<client.equity<<" from "
                    <<order_book_finder[client.equity].sell.top().name<<" for $"<<match_price<<"/share"<<endl;
        }

        if(getopt.transfers)
        {
            transf[order_book_finder[client.equity].buy.top().name].num_bought += num_shares;
            transf[order_book_finder[client.equity].buy.top().name].net_traded -= (match_price * num_shares);
            transf[order_book_finder[client.equity].sell.top().name].net_traded += (match_price * num_shares);
            transf[order_book_finder[client.equity].sell.top().name].num_sold += num_shares;
        }

        temp_client = order_book_finder[client.equity].sell.top();

            order_book_finder[client.equity].sell.pop(); 
            temp_client.quantity -= num_shares;

            if(temp_client.quantity > 0) //if seller still has shares
                order_book_finder[client.equity].sell.push(temp_client);

        temp_client = order_book_finder[client.equity].buy.top();

            order_book_finder[client.equity].buy.pop(); 
            temp_client.quantity -= num_shares;

            if(temp_client.quantity > 0) //if seller still has shares
                order_book_finder[client.equity].buy.push(temp_client);

        //after trade calculated, match price SHOULD CHANGE BY REFERENCE***
        if(getopt.summary)
        {
            commission = ((match_price*num_shares)/100);
            final_summary.total_commission += (2*commission);
            final_summary.total_money_transferred += (match_price*num_shares);
            final_summary.total_comp_trades += 1;
            final_summary.total_shares_traded += num_shares;
        }

    //set up calculation for streaming median
        if(getopt.median or getopt.insider)
        {
            if(order_book_finder[client.equity].min_heap.empty() && order_book_finder[client.equity].max_heap.empty())
                order_book_finder[client.equity].min_heap.push(match_price);

            else if(order_book_finder[client.equity].max_heap.empty())
            {
                if(match_price < order_book_finder[client.equity].min_heap.top()) //max heap should start with lower value
                    order_book_finder[client.equity].max_heap.push(match_price);

                else //add the greater value to min heap, popping first elt
                {
                    order_book_finder[client.equity].max_heap.push(order_book_finder[client.equity].min_heap.top());
                    order_book_finder[client.equity].min_heap.pop();
                    order_book_finder[client.equity].min_heap.push(match_price);
                }
            }
            else 
            {
                if(match_price < order_book_finder[client.equity].max_heap.top())
                    order_book_finder[client.equity].max_heap.push(match_price);
                else
                    order_book_finder[client.equity].min_heap.push(match_price);

                if(order_book_finder[client.equity].min_heap.size() < (order_book_finder[client.equity].max_heap.size() - 1))
                {
                    order_book_finder[client.equity].min_heap.push(order_book_finder[client.equity].max_heap.top());
                    order_book_finder[client.equity].max_heap.pop();
                }

                else if(order_book_finder[client.equity].max_heap.size() < (order_book_finder[client.equity].min_heap.size() - 1))
                {
                    order_book_finder[client.equity].max_heap.push(order_book_finder[client.equity].min_heap.top());
                    order_book_finder[client.equity].min_heap.pop();
                }
            }
        }
    if(order_book_finder[client.equity].buy.empty() || order_book_finder[client.equity].sell.empty())
        return;
    }
}

void make_full_trade(unordered_map<string, Transfer> &transf, Summarize &final_summary, long long &curr_timestamp, 
                    Getopt &getopt, Client &client, unordered_map<string, Order_Book> &order_book_finder, unordered_map<string, Time_Traveler> &timer, set<string> &sorted_keys)
{
    long long match_price = 0;
    long long insider_check = 0;

    //calculate trades
     calculate_trade(transf, final_summary, curr_timestamp, match_price, client, insider_check, getopt, order_book_finder, timer, sorted_keys); 
     //calculate without insiders

    //INSIDER LOGIC
    if(getopt.insider)
    { 
        //long long insider_id = 0;
        if(getopt.insider_set.find(client.equity) != getopt.insider_set.end()) //compare with recently traded equity-> client.equity
        {
            //PERFORM INSIDER LOGIC WITH PROFITS ETC, PERFORM BUY THEN SELL, this is a new order each time
            string insider_string;
            insider_string = "INSIDER_" + client.equity;
            Client insider_client;

            insider_check = get_median(curr_timestamp, true, order_book_finder, client, getopt, sorted_keys); //true, don't prlong long median
            //cout<<"insider check median is "<<insider_check<<endl;
            if(!order_book_finder[client.equity].sell.empty() && insider_check != 0)
            {//cout<<"here"<<endl;
                if((insider_check - order_book_finder[client.equity].sell.top().price) > (0.1 * insider_check))//profits on per share basis
                { 
                insider_client.timestamp = curr_timestamp;
                insider_client.name = insider_string;
                insider_client.buy = true; //if not buy, sell
                insider_client.equity = client.equity;
                insider_client.price = order_book_finder[client.equity].sell.top().price; 
                insider_client.quantity = order_book_finder[client.equity].sell.top().quantity;
                insider_client.id = 0;
                    //cout<<"Insider buyer"<<endl;
                    calculate_trade(transf, final_summary, curr_timestamp, match_price, insider_client, 
                                                                insider_check, getopt, order_book_finder, timer, sorted_keys);
                }
            }

            if(!order_book_finder[client.equity].buy.empty() && insider_check != 0)
            {//os<<"here"<<order_book_finder[client.equity].buy.top().price<<endl;
                if((order_book_finder[client.equity].buy.top().price - insider_check) > (0.1 * insider_check))
                {
                insider_client.timestamp = curr_timestamp;
                insider_client.name = insider_string;
                insider_client.buy = false; //if not buy, sell
                insider_client.equity = client.equity;
                insider_client.price = order_book_finder[client.equity].buy.top().price; 
                insider_client.quantity = order_book_finder[client.equity].buy.top().quantity;
                insider_client.id = 0;
                    //cout<<"Insider seller"<<endl;
                    calculate_trade(transf, final_summary, curr_timestamp, match_price, insider_client,
                                                            insider_check, getopt, order_book_finder, timer, sorted_keys);
                }
            }
        }
    }
}

void read_input(Getopt &getopt)
{
    unordered_map<string, Order_Book> order_book_finder;

    unordered_map<string, Time_Traveler> timer;

    set<string> sorted_keys;
    

    string mode; string client_name;
    string buy_sell; string equity_symbol; long long id = 1;

    long long prevstamp = -1;
    long long curr_timestamp = 0;

    //for summary
    Summarize final_summary;
    final_summary.total_commission = 0;
    final_summary.total_money_transferred = 0;
    final_summary.total_comp_trades = 0;
    final_summary.total_shares_traded = 0;

    //for transfers
    unordered_map<string, Transfer> transf;
    if(getopt.insider && getopt.transfers) //in case insider doesn't perform a trade
    {
        for(auto it = getopt.insider_set.begin(); it != getopt.insider_set.end(); ++it)
        {
            transf["INSIDER_" + *it].num_sold = 0;
            transf["INSIDER_" + *it].num_bought = 0;
            transf["INSIDER_" + *it].net_traded = 0;
        }
    }

    Client new_client;
    new_client.buy = true;

    cin>>mode;

    if(strcmp(mode.c_str(), "TL") == 0)
    {
        string str;
        string timestamp; string price; string quantity;
        
        cin.ignore();
        while(getline(cin, str) && !str.empty())
        { 
            if(str.empty())
                break;
            if(str[0] == '\n')
                break;

            istringstream ss(str);
            ss>>timestamp>>client_name>>buy_sell>>equity_symbol>>price>>quantity;

            if(prevstamp != -1)
            {
                if(stoll(timestamp) < prevstamp)
                    exit(1);
            }

            prevstamp = stoll(timestamp);

            //error checking
            if(stoll(timestamp) < 0 || timestamp.find('.') != timestamp.npos) 
                exit(1);
            else if(price.find('$') != 0 || quantity.find('#') != 0)
                exit(1);
            else if(stoll(price.substr(1)) < 1 || price.find('.') != price.npos)
                exit(1);
            else if(strcmp(buy_sell.c_str(), "BUY") != 0 && strcmp(buy_sell.c_str(), "SELL") != 0)
                exit(1);

            else
            { 
                if(equity_symbol.length() > 4)
                    exit(1);

                unsigned long long i = 0;
                for(i = 0; i < equity_symbol.length(); ++i)
                {
                    if(!isalnum(equity_symbol[i]) && equity_symbol[i] != '_' &&
                        equity_symbol[i] != '.')
                        exit(1);
                }

                for(i = 0; i < client_name.length(); ++i)

                    if(!isalnum(client_name[i]) && client_name[i] != '_')
                        exit(1);

            }

            new_client.timestamp = (stoll(timestamp));
            new_client.name = client_name;
            new_client.equity = equity_symbol;
            new_client.price = stoll(price.substr(1));
            new_client.quantity = stoll(quantity.substr(1)); 
            if(strcmp(buy_sell.c_str(), "SELL") == 0)
                new_client.buy = false;  
            else
                new_client.buy = true;

            new_client.id = id;
            
            if(getopt.transfers && transf.find(new_client.name) == transf.end())
            {
                transf[new_client.name].num_sold = 0;
                transf[new_client.name].num_bought = 0;
                transf[new_client.name].net_traded = 0;
            }
            make_full_trade(transf, final_summary, curr_timestamp, getopt, new_client, order_book_finder, timer, sorted_keys);

            ++id;
        }

    }

    else if(strcmp(mode.c_str(), "PR") == 0)
    {
        string random; unsigned seed;
        string number; unsigned orders;
        string last; char client;
        string latest; char equity;
        string arrival; double rate;

        cin>>random>>seed>>number>>orders>>last>>client>>latest>>equity>>arrival>>rate;

        long long generator_timestamp = 0;
        long long timestamp = 0;
        long long  price = 0;
        long long quantity = 0;


        std::mt19937 gen(seed);
        std::uniform_int_distribution<char> clients('a', client);
        std::uniform_int_distribution<char> equities('A', equity);
        std::exponential_distribution<> arrivals(rate);
        std::bernoulli_distribution buy_or_sell;
        std::uniform_int_distribution<> prices(2,11);
        std::uniform_int_distribution<> quantities(1,30);

        for(unsigned int i = 0; i < orders; ++i)
        {
            timestamp = generator_timestamp;
            generator_timestamp = generator_timestamp + floor(arrivals(gen) + 0.5);
            client_name = string("C_") + clients(gen);
            buy_sell = (buy_or_sell(gen) ? "BUY" : "SELL");
            equity_symbol = string("E_") + equities(gen);
            price = 5 * prices(gen);
            quantity = quantities(gen);

        

            new_client.timestamp = timestamp;
            new_client.name = client_name; 
            new_client.equity = equity_symbol; 
            new_client.price = price; 
            new_client.quantity = quantity;
            if(strcmp(buy_sell.c_str(), "SELL") == 0) 
                new_client.buy = false;
            else 
                new_client.buy = true; 

            new_client.id = id; 

            if(getopt.transfers && transf.find(new_client.name) == transf.end())
            {
                transf[new_client.name].num_sold = 0;
                transf[new_client.name].num_bought = 0;
                transf[new_client.name].net_traded = 0;
            }

            make_full_trade(transf, final_summary, curr_timestamp, getopt, new_client, order_book_finder, timer, sorted_keys);

            ++id;        
        }    
    }

        if(getopt.median)
        {
            long long insider_check = get_median(curr_timestamp, false, order_book_finder, new_client, getopt, sorted_keys);
            if(insider_check != 0)
                {insider_check = 0;}
        }
    

os<< "---End of Day---"<<endl; //timestamp has moved on, no more PR or TL input

    if (getopt.summary)
    {
        os<<"Commission Earnings: $"<<final_summary.total_commission<<endl;
        os<<"Total Amount of Money Transferred: $"<<final_summary.total_money_transferred<<endl;
        os<<"Number of Completed Trades: "<<final_summary.total_comp_trades<<endl;
        os<<"Number of Shares Traded: "<<final_summary.total_shares_traded<<endl;
    }

    if(getopt.transfers)
    {
        //use ordered set or vector for transfers output
        vector<string> sorted_keys;
        for(auto it = transf.begin(); it != transf.end(); ++it)
        {
            sorted_keys.push_back(it->first);
        }
        sort(sorted_keys.begin(), sorted_keys.end());

        for(auto it = sorted_keys.begin(); it!= sorted_keys.end(); ++it)
        {
            os<<*it<<" bought "<<transf[*it].num_bought<<" and sold "<<transf[*it].num_sold
                                                    <<" for a net transfer of $"<<transf[*it].net_traded<<endl;
        }
    }

    if(getopt.ttt)
    {

        for(auto it = getopt.ttt_set.begin(); it != getopt.ttt_set.end(); ++it)
        { //cout<<"it is "<<*it<<endl;

    if(timer[*it].curr.amt_sell == -1 && timer[*it].curr.amt_buy == -1)
    os<<"Time travelers would buy "<<*it<<" at time: "<<timer[*it].curr.time_sell<<" and sell it at time: "<<timer[*it].curr.time_buy<<endl;

    else if(timer[*it].curr.amt_buy == -1 && timer[*it].curr.amt_sell != -1)
    os<<"Time travelers would buy "<<*it<<" at time: "<<timer[*it].prev.time_sell<<" and sell it at time: "<<timer[*it].prev.time_buy<<endl;

    else if(((timer[*it].curr.amt_buy - timer[*it].curr.amt_sell) > (timer[*it].prev.amt_buy - timer[*it].prev.amt_sell)) || 
                ((timer[*it].prev.amt_sell == -1) || (timer[*it].prev.amt_buy == -1)))
    os<<"Time travelers would buy "<<*it<<" at time: "<<timer[*it].curr.time_sell<<" and sell it at time: "<<timer[*it].curr.time_buy<<endl;

    else if(((timer[*it].prev.amt_buy - timer[*it].prev.amt_sell) > (timer[*it].curr.amt_buy - timer[*it].curr.amt_sell)) || 
            ((timer[*it].curr.amt_buy - timer[*it].curr.amt_sell) == (timer[*it].prev.amt_buy - timer[*it].prev.amt_sell)))
    os<<"Time travelers would buy "<<*it<<" at time: "<<timer[*it].prev.time_sell<<" and sell it at time: "<<timer[*it].prev.time_buy<<endl;

        }
    }//end getopt ttt
}//end read_input




int main(int argc, char** argv)
{
    ios_base::sync_with_stdio();

struct option long0pts[] = {

    { "summary", no_argument, NULL, 's' },
    { "verbose", no_argument, NULL, 'v' },
    { "median", no_argument, NULL, 'm' },
    { "transfers", no_argument, NULL, 't' },
    { "insider", required_argument, NULL, 'i' },
    { "ttt", required_argument, NULL, 'g' },
    { "help", no_argument, NULL, 'h' }
                            };

    opterr = false;
    bool summary = false; bool verbose = false;
    bool median = false; bool transfers = false;
    bool insider = false; bool ttt = false; //these can appear more than once

    set<string> set1;
    vector<string> set2;
    int opt = 0, index = 0;

    while((opt = getopt_long(argc, argv, "svmti:g:h", long0pts, &index)) != -1)
    {

        switch (opt) 
        {
        case 's':
            summary = true;
            break;
        case 'v':
            verbose = true;
            break;
        case 'm':
            median = true;
            break;
        case 't':
            transfers = true;
            break;
        case 'i':
            insider = true;
            set1.insert(cpp_str(optarg));
            
            break;
        case 'g':
            ttt = true;
            set2.push_back(cpp_str(optarg));
            
            break;
        case 'h':
            cerr<<"This program pairs buyers and sellers in an electronic stock exchange market"<<endl;
            exit(0);
            break;

        default:
            cerr<< "I didn't recognize one of your flags!"<<endl;
            exit(1);
        }
    }

Getopt getopt;
getopt.summary = summary;
getopt.verbose = verbose;
getopt.median = median;
getopt.transfers = transfers;
getopt.insider = insider;
getopt.ttt = ttt;
getopt.insider_set = set1;
getopt.ttt_set = set2;

read_input(getopt); //fills buy/sell queues with clients

cout<<os.str();
    return 0;
}
