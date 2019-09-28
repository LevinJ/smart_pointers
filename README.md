# CPPND: Memory Management Chatbot

Initial project codes have been modified as required to meet project rubric. Below are the detailed descriptions about relevant code changes.

## Exclusive Ownership

### `_chatLogic` ownership

`_chatLogic` is now owned exclusively by `ChatbotPanelDialog`. This is achieved by using unique smart pointer `std::unique_ptr<ChatLogic> _chatLogic;`.

### `GraphNode` ownership

`GraphNode` instance in the vector `_nodes` are execlusivly owned by `ChatLogic`. This is realized by use of vector of unqiue smart pointer `std::vector<std::unique_ptr<GraphNode>> _nodes;`

### `GraphNode` ownership is not transferred

GraphNode instance is  exclusvisely owned by `ChatLogic` in the form of `unique_ptr`. When passing it in function like `std::find_if`, refence is used, something like below,

```
auto parentNode = std::find_if(_nodes.begin(), _nodes.end(), [&parentToken](std::unique_ptr<GraphNode> &node) { return node->GetID() == std::stoi(parentToken->second); });
auto childNode = std::find_if(_nodes.begin(), _nodes.end(), [&childToken](std::unique_ptr<GraphNode> &node) { return node->GetID() == std::stoi(childToken->second); });
```

### outgoing edges `_childEdges` ownership

GraphNodes exclusively own the outgoing GraphEdges and hold non-owning references to incoming GraphEdges.

This is mainly acheived by using both `unique_ptr` smart pointer and raw pointer. 

```
// data handles (owned)
std::vector<std::unique_ptr<GraphEdge>> _childEdges;  // edges to subsequent nodes

// data handles (not owned)
std::vector<GraphEdge *> _parentEdges; // edges to preceding nodes 
```

An alternative approach is to use a combination of `shared_ptr` and `weak_ptr`, but it has two disadvantage.

1. The concept of exclusive ownership for outgoing edges are weakend.
2. Code changes will be much more complicated, and are spread in more files.

## The Rule Of Five



### Rule of Five implementation

Chatbot class's Rule of Five implementation is as below. In particular, copy and swap idom is applied to implement the copy assignment and move assignment operator, in an attempt to avoid code duplication and provide a strong exception guarantee. 

```
//member for member , shallow swap
void swap(ChatBot& first, ChatBot& second){
	using std::swap;
	swap(first._image, second._image);
	swap(first._currentNode, second._currentNode);
	swap(first._rootNode, second._rootNode);
	swap(first._chatLogic, second._chatLogic);
}

//copy constructor
ChatBot::ChatBot(const ChatBot& other){
	std::cout << "ChatBot Copy Constructor" << std::endl;
	_image = new wxBitmap(*other._image);
	_currentNode = other._currentNode;
	_rootNode = other._rootNode;
	_chatLogic = other._chatLogic;
};

//copy assignment operator, use copy and swap idom
ChatBot& ChatBot::operator=(const ChatBot& other){
	std::cout << "ChatBot Copy Assignment" << std::endl;
	ChatBot copy(other);
	swap(*this, copy);
	return *this;
}

//move constructor
//ChatBot::ChatBot(ChatBot&& other):ChatBot(){
ChatBot::ChatBot(ChatBot&& other){
	std::cout << "ChatBot Move Constructor" << std::endl;
	 _image = nullptr;
	_chatLogic = nullptr;
	_rootNode = nullptr;
	swap(*this, other);
}

//move assignment operator
ChatBot& ChatBot::operator=(ChatBot&& other){
	std::cout << "ChatBot Move Assignment Operator" << std::endl;
	swap(*this,other);
	return *this;
}
```

### printing indicates Rule of five components

After each query is entered, console output is as below,

```
ChatBot Constructor
ChatBot Move Constructor
ChatBot Move Assignment Operator
ChatBot Destructor
ChatBot Destructor 
```


## Moving Semantics


### move local `Chatbot` instance to root node

As we intend to pass `Chatbot` instance among current nodes, a local `Chatbot` instance is first created on stack and passed to rootnode.

```
// create instance of chatbot
ChatBot chatbot_temp("../images/chatbot.png");

// add pointer to chatlogic so that chatbot answers can be passed on to the GUI
chatbot_temp.SetChatLogicHandle(this);
// add chatbot to graph root node
chatbot_temp.SetRootNode(rootNode);
rootNode->_chatBot.reset(new ChatBot());
rootNode->MoveChatbotHere(std::move(chatbot_temp));
```

Note that the line `rootNode->_chatBot.reset(new ChatBot());` is inserted to create below workflow as requried by project rubric.

```
ChatBot Constructor
ChatBot Move Constructor
ChatBot Move Assignment Operator
ChatBot Destructor
ChatBot Destructor 
```

Under the hood, here is what happens.

1. Create an empty Chatbot instance on the root node,  `rootNode->_chatBot.reset(new ChatBot());`  
This outputs `ChatBot Constructor`
2. Call `MoveChatbotHere`, `rootNode->MoveChatbotHere(std::move(chatbot_temp))`  
This outputs `ChatBot Move Constructor3.`  
`MoveChatbotHere` is as below
```
void GraphNode::MoveChatbotHere(ChatBot chatbot)
{
    *_chatBot = std::move(chatbot);
    _chatBot->_chatLogic->SetChatbotHandle(_chatBot.get());
    _chatBot->SetCurrentNode(this);

}
```
3. Move the temporary ChatBot instance to the root node, with above line `*_chatBot = std::move(chatbot);`  
This outputs `ChatBot Move Assignment Operator`

4. The temporary Chatbot instance deallocate  
This outputs `ChatBot Destructor`

5. The local stack Chatbot instance deallocate  

This outputs `ChatBot Destructor`



### `_ChatLogic` holds non-owning references to `Chatbot`

ChatLogic has no ownership relation to the ChatBot instance and thus is no longer responsible for memory allocation and deallocation.
Each time the chatbot instance move to a new node, we would update chatbot refence for the Chatlogic.This is implemented in `GraphNode`, the line `_chatBot->_chatLogic->SetChatbotHandle(_chatBot.get());`.  

```
void GraphNode::MoveChatbotHere(ChatBot chatbot)
{
    *_chatBot = std::move(chatbot);
    _chatBot->_chatLogic->SetChatbotHandle(_chatBot.get());
    _chatBot->SetCurrentNode(this);

}
```

### move `_GraphEdge` instance to `GraphNode`
This is done by below codes,

```
// create new edge
std::unique_ptr<GraphEdge> edge(new GraphEdge(id));
edge->SetChildNode((*childNode).get());
edge->SetParentNode((*parentNode).get());
_edges.push_back(edge.get());

// find all keywords for current node
AddAllTokensToElement("KEYWORD", tokens, *edge);

// store reference in child node and parent node
(*childNode)->AddEdgeToParentNode(edge.get());
(*parentNode)->AddEdgeToChildNode(std::move(edge));
```
