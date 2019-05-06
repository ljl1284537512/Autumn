## 如何遍历二叉树（广度/深度/递归）

```java
package com.example.printtree;

import java.util.ArrayDeque;
import java.util.Queue;
import java.util.Stack;

/**
 * 二叉树打印技巧
 *
 * @author lijianlong
 * @since 2019/5/6
 */
public class TreePrintTest {

    /**
     * 数据初始化
     *
     * @return
     */
    public static Node<String> init() {
        Node<String> root = new Node<>("0");
        root.left = new Node<>("1");
        root.right = new Node<>("2");
        root.left.left = new Node<>("3");
        root.left.right = new Node<>("4");
        root.left.right.left = new Node<>("6");
        root.left.right.right = new Node<>("7");
        root.right.right = new Node<>("5");
        return root;
    }

    /**
     * 递归深度遍历
     * 缺点是在节点数目较多的情况下，会导致栈溢出
     *
     * @param root
     */
    public void recursionPrint(Node root) {
        System.out.println(root.t);
        if (root.left != null) {
            recursionPrint(root.left);
        }
        if (root.right != null) {
            recursionPrint(root.right);
        }
    }

    /**
     * 广度优先遍历
     * 描述：利用队列结构，将每个节点查出来的左右字节点先放进队列中，然后从队列中一个一个往外面取，一边遍历，一边存入新的左右子节点
     * 其实，如果不是普通集合在遍历过程中不支持删减/增加元素，普通集合也能代替队列做数据存取。
     *
     * @param root
     */
    public void rangePrint(Node root) {
        Queue<Node> preViewNodes = new ArrayDeque<>();
        preViewNodes.add(root);
        while (!preViewNodes.isEmpty()) {
            Node next = preViewNodes.poll();
            Node left = next.left;
            if (left != null) {
                System.out.println(left.t);
                preViewNodes.add(left);
            }
            Node right = next.right;
            if (right != null) {
                System.out.println(right.t);
                preViewNodes.add(right);
            }
        }
    }

    /**
     * 深度优先遍历
     * 描述: 利用栈的结构特点，每次从栈中拿出节点的时候，先打印值，再将其左右节点依次放入栈中。下次再从栈中获取的时候，其实是刚放进去的子节点
     * 注意点：因为栈的顺序是先进后出，所以要先压入右节点
     *
     * @param root
     */
    public void deepPrint(Node root) {
        Stack<Node> preViewNodes = new Stack<>();
        preViewNodes.push(root);

        while (!preViewNodes.isEmpty()) {
            Node pop = preViewNodes.pop();
            System.out.println(pop.t);
            Node right = pop.right;
            if (right != null) {
                preViewNodes.push(right);
            }
            Node left = pop.left;
            if (left != null) {
                preViewNodes.push(left);
            }
        }
    }

    public static void main(String[] args) {
        Node<String> root = init();
        new TreePrintTest().rangePrint(root);
    }
}

```
