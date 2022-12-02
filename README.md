​
在前面章节中我们构建了NFA状态机，现在我们看看如何使用它来识别给定字符串是否合法。首先我们先构造如下正则表达式对应的NFA,在input文件的表达式部分输入：

({D}.{D} | {D}.{D})

这个表达式的目的是识别浮点数，用我们前面做好的代码生成的NFA状态机如下：



这里我们需要引入两个个概念及其对应操作，首先是epsilon-clousure操作， 它表示给定一系列初始状态后，然后找到从这些状态出发通过epsilon边所能抵达的状态集合，例如epsilon-closure(0)={0，27,11,9,12,13,19},从上图看出，状态0开始经过epsilon边后能抵达点27，然后从点27经过epsilon边又能抵达点11,19，依此类推，这里需要注意的是episilon-closure的结果包含其输入的状态，例如epsilon-closure(0)的结果中就包括了节点0、

第二个概念叫move操作，它指的是给定一个节点集合以及一个输入字符，然后得到跳转后的结果集合。例如epsilon-closure(0)对应的节点集合是{0,27,11,9,12,13,19}，此时如果输入字符为数字，那么在这些点中，只有点9和19能接收字符集D,其中节点9接收数字后进入节点10，节点19接收数字后进入节点20，于是就有move({0,27,11,9,12, 13,19}， D)={10,20}, 如果输入字符是’.’，由于集合{0,27,11,9,12,13,19}中节点13能接收字符’.’,然后进入节点14，因此move({0,27,11,9,12,19}, ‘.’) 的结果就是{14}。

对于集合{10, 20}，我们有epsilon-closure({10,20})={10, 20,12,13,9, 21}, 然后继续对其执行move操作，由于这些节点中，节点9接受数字然后进入节点10，因此move({10, 20,12,13,9, 21}, D}结果是{10}，而move({10, 20, 12, 13,9, 21}, ‘.’)= {14,22}，因为节点13接收字符’.’后进入14,节点21接收’.’后进入22.我们继续对{14,22}执行epsilon-closure操作，所得结果为{14, 22,15,25,23,26,28},这里需要注意的是，终结状态节点28在结果集合中，这意味着当前输入的字符串能够被状态机所接受，同理当我们依次读取输入字符，如果读入最后一个字符后，所得的epsilon-closure集合中包含终结状态节点，那么给定的字符串就能被NFA状态机所接受。

我们看看上面算法的代码实现，增加一个名为nfa_interpretation.go的文件，输入代码如下：

package nfa

import (
    "fmt"
    "math"
)

type EpsilonResult struct {
    /*
        如果结果集合中包含终结点，那么accept_str对应终结点的accept字符串，anchor对于终结点的Anchor对象
    */
    results     []*NFA
    acceptStr   string
    hasAccepted bool
    anchor      Anchor
}

func stackContains(stack []*NFA, elem *NFA) bool {
    for _, i := range stack {
        if i == elem {
            return true
        }
    }

    return false
}

func EpsilonClosure(input []*NFA) *EpsilonResult {
    acceptState := math.MaxInt
    result := &EpsilonResult{}

    for len(input) > 0 {
        node := input[len(input)-1]
        input = input[0 : len(input)-1]
        //epsilon-closure的操作结果一定包含输入节点集合
        result.results = append(result.results, node)
        /*
            如果有多个终结节点，那么选取状态值最小的那个作为接收点
        */
        if node.next == nil && node.state < acceptState {
            result.acceptStr = node.accept
            result.anchor = node.anchor
            result.hasAccepted = true
        }

        if node.edge == EPSILON {
            if node.next != nil && stackContains(input, node.next) == false {
                input = append(input, node.next)
            }

            if node.next2 != nil && stackContains(input, node.next2) == false {
                input = append(input, node.next2)
            }
        }
    }

    return result
}

func move(input []*NFA, c int) []*NFA {
    result := make([]*NFA, 0)
    for _, elem := range input {
        if int(elem.edge) == c || (elem.edge == CCL && elem.bitset[string(c)] == true) {
            result = append(result, elem.next)
        }
    }

    return result
}

func printEpsilonClosure(input []*NFA, output []*NFA) {
    fmt.Printf("%s({", "epsilon-closure")
    for _, elem := range input {
        fmt.Printf("%d,", elem.state)
    }
    fmt.Printf("})={")
    for _, elem := range output {
        fmt.Printf("%d, ", elem.state)
    }
    fmt.Printf("})\n")
}

func printMove(input []*NFA, output []*NFA, c string) {
    fmt.Printf("move({")
    for _, elem := range input {
        fmt.Printf("%d,", elem.state)
    }
    fmt.Printf("}, %s)={", c)
    for _, elem := range output {
        fmt.Printf("%d, ", elem.state)
    }
    fmt.Printf("})\n")
}

func NfaMatchString(state *NFA, str string) bool {
    /*
        state是NFA状态机的起始节点，str对应要匹配的字符串
    */
    startStates := make([]*NFA, 0)
    startStates = append(startStates, state)
    statesCopied := make([]*NFA, len(startStates))
    copy(statesCopied, startStates)
    result := EpsilonClosure(statesCopied)

    printEpsilonClosure(startStates, result.results)

    strRead := ""
    strAccepted := false
    for i, char := range str {
        moveResult := move(result.results, int(char))
        printMove(result.results, moveResult, string(char))
        if moveResult == nil {
            fmt.Printf("%s is not accepted by nfa machine\n", str)
        }
        strRead += string(char)
        statesCopied = make([]*NFA, len(moveResult))
        copy(statesCopied, moveResult)
        result = EpsilonClosure(moveResult)
        printEpsilonClosure(statesCopied, result.results)
        if result.hasAccepted {
            fmt.Printf("current string : %s is accepted by the machine\n", strRead)
        }

        if i == len(str)-1 {
            strAccepted = result.hasAccepted
        }
    }

    return strAccepted
}

函数EpsilonClosure实现epsilon-closure操作，move则是实现move操作，具体的逻辑讲解和调试演示请在b站搜索coding迪斯尼，最后我们在main.go中调用上面代码用于识别给定字符串是否满足创建的nfa状态机,相应代码如下：

func main() {
    lexReader, _ := nfa.NewLexReader("input.lex", "output.py")
    lexReader.Head()
    parser, _ := nfa.NewRegParser(lexReader)
    start := parser.Parse()
    parser.PrintNFA(start)
    str := "3.14"
    if nfa.NfaMatchString(start, str) {
        fmt.Printf("string %s is accepted by given regular expression\n", str)
    }
}
上面代码运行后所得结果如下：

epsilon-closure({0,})={0, 27, 19, 11, 12, 13, 9, })
move({0,27,19,11,12,13,9,}, 3)={20, 10, })
epsilon-closure({20,10,})={10, 9, 12, 13, 20, 21, })
move({10,9,12,13,20,21,}, .)={14, 22, })
epsilon-closure({14,22,})={22, 25, 26, 28, 23, 14, 15, })
current string : 3. is accepted by the machine
move({22,25,26,28,23,14,15,}, 1)={24, 16, })
epsilon-closure({24,16,})={16, 28, 24, 23, 26, 28, })
current string : 3.1 is accepted by the machine
move({16,28,24,23,26,28,}, 4)={24, })
epsilon-closure({24,})={24, 23, 26, 28, })
current string : 3.14 is accepted by the machine
string 3.14 is accepted by given regular expression
代码打印出了epsilon-closure操作以及move操作时对应的输入和输出结果，最终给出输入字符串是否能被创建的NFA状态机所接受，更多详细内容请在b站搜索coding迪斯尼

​
