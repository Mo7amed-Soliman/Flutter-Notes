

void main() {

  List<String> fruits = ['apple', 'banana'];

  List<String>? moreFruits = null;

  

  List<String> allFruits = [...fruits, ...?moreFruits];

  

  print(allFruits); // Output: [apple, banana, orange, grape]

}