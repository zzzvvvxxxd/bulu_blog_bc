title: Trie树
date: 2016-04-06 13:28:11
categories: 多线程系列
tags: [数据结构,算法]
---

## Trie树模板
题目来源：[http://hihocoder.com/problemset/problem/1014](http://hihocoder.com/problemset/problem/1014)  

```C++
#include <iostream>
#include <string>
using namespace std;

enum NODE_TYPE {
    COMPLETED,
    UNCOMPLETED
};

class TrieNode {
public:
    TrieNode* childNodes[26];
    int freq;
    char nodeChar;
    enum NODE_TYPE type;
    TrieNode(char ch) {
        this->nodeChar = ch;
        this->freq = 0;
        for(int i = 0; i < 26; i++) {
            this->childNodes[i] = NULL;
        }
        this->type = UNCOMPLETED;
    }
    void add_freq();
    void set_type(enum NODE_TYPE);
};

class Trie {
public:
    TrieNode* root;
    Trie() {
        root = new TrieNode(' ');
    }
    void insert(string str);
    bool find(string str);
    int query(string str);
};

int charToIndex(char ch) {
    return ch - 'a';
}

void Trie::insert(string str) {
    TrieNode *current_node = this->root;
    for(int i = 0; i < str.length(); i++) {
        int index = charToIndex(str[i]);
        char ch = str[i];
        if(current_node->childNodes[index] == NULL) {
            current_node->childNodes[index] = new TrieNode(ch);
        }

        current_node->childNodes[index]->add_freq();

        if(i == str.length()-1) {
            current_node->childNodes[index]->set_type(COMPLETED);
        } else if(current_node->childNodes[index]->type != COMPLETED){
            current_node->childNodes[index]->set_type(UNCOMPLETED);
        }
        current_node = current_node->childNodes[index];
    }
}

bool Trie::find(string str) {
    TrieNode *current_node = this->root;
    int i = 0;
    while(i < str.length()) {
        char ch = str[i];
        int index = charToIndex(ch);
        if(current_node->childNodes[index] == NULL) {
            break;
        }
        i++;
        current_node = current_node->childNodes[index];
    }
    cout << current_node->nodeChar << " " << current_node->type << endl;
    return current_node->type == COMPLETED && i == str.length();
}

int Trie::query(string str) {
    TrieNode *current_node = this->root;
    int i = 0;
    int count;
    while(i < str.length()) {
        char ch = str[i];
        int index = charToIndex(ch);
        if(current_node->childNodes[index] == NULL) {
            break;
        }
        i++;
        current_node = current_node->childNodes[index];
    }
    if(i == str.length()){
        return current_node->freq;
    }
    return 0;
}

void TrieNode::set_type(enum NODE_TYPE t) {
    this->type = t;
}

void TrieNode::add_freq() {
    this->freq++;
}

int main() {
    Trie trie;
    int loop;
    cin >> loop;
    while(loop > 0) {
        string str;
        cin >> str;
        trie.insert(str);
        loop--;
    }
    cin >> loop;
    while(loop > 0) {
        string str;
        cin >> str;
        cout << trie.query(str) << endl;
        loop--;
    }
    return 0;
}
```