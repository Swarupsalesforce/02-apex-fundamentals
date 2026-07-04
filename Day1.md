Question -1
Variable with your age → print "My age is 29"
ans-->
integer age= 29;
System.debug('My age is ' +age);
Question -2
Two numbers → print sum, difference, product
ans-->
integer a= 29;
Integer b= 10;
System.debug('sum ' +(a+b));
System.debug('Diffrence '+(a-b));
System.debug('Product '+(a*b));
Question -3
Price 999.99 → which type? → print it doubled
Ans-->
Decimal price= 999.9;
System.debug('Price Doubled ' +(price*2)
Question -4
firstName + lastName Strings → print full name with a space

Ans-->
String firstName= 'Swarup';
String lastName = 'Satapathy';
System.debug('Full Name - '+ firstName+ ' '+lastName);
Question -5-Number → 'Even' or 'Odd' (Math.mod(n, 2))
Ans-
integer N= 21;
integer r=Math.mod(N, 2);
if(r==0){
    system.debug('Even Number');
}else{
    System.debug('Odd Number');
}
Q- Number → 'Positive' / 'Negative' / 'Zero' (if / else if / else)

Ans-
integer N= 21;
if(n>0){
    system.debug('Positive Number');
}else if(n<0){
    System.debug('Negative Number');
}else{
    System.debug('Zero');
}

Q-Score 0–100 → 'Pass' ≥50 else 'Fail' — decide what exactly 50 does, comment your rule

Ans-
integer score= 45;
if(score>=50 && Score<=100){
    system.debug('Pass');
}else if(score <50 && score >=0){
    system.debug('Fail');
    
}else{
    System.debug('Invalid Score');
}

Q-Two numbers → print the bigger, no Math.max

Ans-
integer a= 45;
integer b= 25;
if(a>b){
    system.debug('Bigger No- '+a);
}else{
    system.debug('Bigger No- '+b);
    
}

Q-Year → 'Leap year' if divisible by 4, else 'Not'

Ans-
integer y= 2021;
if((Math.mod(y,4)==0 && Math.mod(y,100)!=0)||Math.mod(y,400)==0){
    system.debug('Leap Year- '+y);
}else{
    system.debug('Not a Leap Year- '+y);
}

Q-Amount → 'Small' <10000 / 'Medium' 10000–100000 / 'Large' >100000
Ans-
integer A= 2021;
if(a<10000 && a>0){
    system.debug('Small');
}else if(a>=10000 && a<=100000){
    system.debug('Medium');
}else if(a>100000){
    system.debug('Large');
}else{
    system.debug('Invalid Amount');
}

Assignment-Script: amount + customerTier ('Gold'/'Silver'/'Bronze') → print discount 20 / 10 / 5 

ans= 
Decimal amount=100;
string customerTier='Platinum';
Decimal discountPercent;
if(customerTier =='Gold'){
    discountPercent=20;
    
}else if(customerTier =='Silver'){
    discountPercent=10;
}else if(customerTier =='Bronze'){
    discountPercent =5;
}else{
    discountPercent=0;
}
System.debug('Tier: '+ customerTier +' , Discount: ' + discountPercent + '%');

