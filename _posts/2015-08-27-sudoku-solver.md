---
title: 数独求解器
layout: post
---

> 数独(Sudoku)是一种非常简单的数字游戏， 对于经典的数独其规则是根据给出的数字在一个9X9的方格中填写剩余方格中的数字，需要保证每一行的数字包含1~9，每一列包含1~9，每个3X3的子方格中也包含1~9。关于数独的各种理论分析这里不做过多的讨论，而只是给出一种简单并且适合计算机求解的方法。

#### 1. 求解思路

> 首先对于一个给出的题目，其可能存在唯一解，可能存在多个解，也可能无解，我们的算法则是在有解的情况下给出其中的一个解答。
>
1. 首先根据已有的数字，利用上面提到的三个约束条件来确定可以唯一确定数字的方格，并将其填入相应的数据
>
2. 对于其他无法确定的方格，选取其中一个，并利用上面提到的三个约束条件找出可能填写的数字，并尝试其中的一个，如此就形成一个新的数独问题，对此数独问题按照求解流程进行递归求解
>
3. 如果递归求解成功，则数独问题求解完成，如果求解失败（如果存在一个方格中没有能够填写的数字或是已经将一个方格中的全部可能的数字都尝试后依然无法完成求解），则继续尝试下一个可能的数字
>
4. 如果全部数字都无法完成求解则此数独无解
>
> 求解的过程是一个典型的深度优先搜索问题，程序中利用栈来保存当前的状态并对每一个分支进行分析，如果分析失败则回溯到最近一次栈中的状态，实际过程中可以利用函数递归调用中的堆栈来进行状态的保存，当然也可以自己实现一个栈的数据结构实现非递归的算法。

#### 2. 程序实现

> 根据以上的求解思路，这里给出一种Java语言的实现，其代码如下
>
```java
import java.io.*;
import java.util.Scanner;
import java.util.ArrayList;
>
class Sudoku {
    private int[][] matrix;
    final int SIZE = 9;
>
    public Sudoku() {
        matrix = new int[SIZE][SIZE];
    }
>
    //复制一个数独，便于新的数独问题的生成
    public Sudoku(Sudoku s) {
        this();
        int size = SIZE;
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                set(i, j, s.get(i, j));
            }
        }
    }
>
    //读写数独方格
    public void set(int i, int j, int v) {
        matrix[i][j] = v;
    }
>
    public int get(int i, int j) {
        return matrix[i][j];
    }
>
    //根据目前的数字，利用约束填写其他的方格直到无法找到可以填写的方格为止
    private boolean calculate() {
        int size = SIZE;
        int n;
        //遍历全部的方法，如果能够找到可以填写的方格，则进行进行遍历知道无法找到为止
        do {
            n = 0;
            for (int i = 0; i < size; i++) {
                for (int j = 0; j < size; j++) {
                    //如果已经填写了数字则不进行分析
                    if (get(i, j) != 0) {
                        continue;
                    }
                    //获取可以填写的数字
                    ArrayList<Integer> nums = getAvailNums(i, j);
                    int s = nums.size();
                    //如果是唯一的，则进行填写，并记录找到一个方格
                    if (s == 1) {
                        int v = nums.get(0);
                        set(i, j, v);
                        n++;
                    }
                    //如果找到一个不能填写任何数字的方格则返回求解失败
                    if (s == 0) {
                        return false;
                    }
                }
            }
        } while (n > 0);
        //遍历完成返回遍历成功
        return true;
    }
>
    //数独求解主流程
    public boolean solve() {
        int size = SIZE;
        //根据约束条件填写方格，如果失败则整个求解流程失败
        if (!calculate()) {
            return false;
        }
        //遍历整个方格没有填写的方格，直到全部填写完成为止
        int n;
        do {
            n = 0;
            for (int i = 0; i < size; i++) {
                for (int j = 0; j < size; j++) {
                    //如果已经填写则增加计数
                    if (get(i, j) != 0) {
                        n++;
                        continue;
                    }
                    //直到当前方格中可以填写的数字
                    ArrayList<Integer> nums = getAvailNums(i, j);
                    int s = nums.size();
                    //如果已经没有可以填写的数字，则返回求解失败
                    if (s == 0) {
                        return false;
                    }
                    //对于每一个可以填写的数字，生成一个新的数独问题，递归进行求解
                    for (int k = 0; k < s; k++) {
                        Sudoku ns = new Sudoku(this);
                        ns.set(i, j, nums.get(k));
                        //如果求解成功，则保存答案，返回成功
                        if (ns.solve()) {
                            for (int p = 0; p < size; p++) {
                                for (int q = 0; q < size; q++) {
                                    set(p, q, ns.get(p, q));
                                }
                            }
                            return true;
                        }
                    }
                    //如果全部尝试后依然无法得到解答，则求解失败
                    return false;
                }
            }
        } while (n != size * size);
        //全部填满则返回求解成功
        return true;
    }
>
    //获取当前方格中可以填写的数字
    public ArrayList<Integer> getAvailNums(int i, int j) {
        int size = SIZE;
        //分析行和列
        ArrayList<Integer> nums = getNums();
        for (int k = 0; k < size; k++) {
            int index;
            int v1 = matrix[i][k];
            if ((index = nums.indexOf(v1)) != -1) {
                nums.remove(index);
            }
            int v2 = matrix[k][j];
            if ((index = nums.indexOf(v2)) != -1) {
                nums.remove(index);
            }
        }
>
        int ssize = (int) Math.sqrt(size);
>
        //分析子方格
        int m = i / ssize;
        int n = j / ssize;
        for (int p = m * ssize; p < (m + 1) * ssize; p++) {
            for (int q = n * ssize; q < (n + 1) * ssize; q++) {
                int index;
                int v = matrix[p][q];
                if ((index = nums.indexOf(v)) != -1) {
                    nums.remove(index);
                }
            }
        }
        return nums;
    }
>
    private ArrayList<Integer> getNums() {
        int size = SIZE;
        ArrayList<Integer> nums = new ArrayList<Integer>();
        for (int k = 0; k < size; k++) {
            nums.add(k + 1);
        }
        return nums;
    }
>
    //从文件加载数独
    public void load(String file) {
        int size = SIZE;
        try {
            BufferedReader reader = new BufferedReader(new FileReader(new File(file)));
            for (int i = 0; i < size; i++) {
                String line = reader.readLine();
                if (line == null) {
                    break;
                }
                Scanner in = new Scanner(line);
                for (int j = 0; j < size; j++) {
                    if (!in.hasNext()) {
                        break;
                    }
                    matrix[i][j] = in.nextInt();
                }
            }
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
>
    //输出数独方格情况
    public void print() {
        int size = SIZE;
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; j++) {
                System.out.printf("%d ", matrix[i][j]);
            }
            System.out.println();
        }
    }
>
    //程序入口
    public static void main(String[] args) {
        Sudoku s = new Sudoku();
        s.load(args[0]);
        s.print();
        System.out.println();
>
        s.solve();
        s.print();
    }
}
```
>

#### 3. 程序运行效果
> 待求解数独，其中0代表需要填写的方格
>
```
0 0 0 0 0 0 0 0 2 
0 0 4 2 0 0 6 0 1 
6 0 0 0 0 0 9 0 0 
9 6 0 8 0 4 1 0 0 
0 0 0 9 0 3 0 0 0 
0 0 8 7 0 6 0 4 9 
0 0 5 0 0 0 0 0 8 
1 0 7 0 0 8 3 0 0 
4 0 0 0 0 0 0 0 0 
```
> 求解后的结果
> 
```
8 1 9 4 6 5 7 3 2 
5 7 4 2 3 9 6 8 1 
6 2 3 1 8 7 9 5 4 
9 6 2 8 5 4 1 7 3 
7 4 1 9 2 3 8 6 5 
3 5 8 7 1 6 2 4 9 
2 3 5 6 7 1 4 9 8 
1 9 7 5 4 8 3 2 6 
4 8 6 3 9 2 5 1 7 
```
>
