# HalsteadMetricForPhp-Delphi-
unit Unit1;

interface

uses
  Windows, Messages, SysUtils, Variants, Classes, Graphics, Controls, Forms,
  Dialogs, StdCtrls, ComCtrls, Menus, StdActns, ActnList, ExtCtrls, Buttons, Math;

type
  TMainFrm = class(TForm)
    mmoInputText: TMemo;
    MainMenu: TMainMenu;
    MenusBtn: TMenuItem;
    OpenBtn: TMenuItem;
    CloseBtn: TMenuItem;
    DlgOpen: TOpenDialog;
    DlgSave: TSaveDialog;
    ActionList: TActionList;
    OpenPhpFile: TAction;
    ClosePhpFile: TAction;
    ControlPanel: TPanel;
    btnOpenFile: TBitBtn;
    btnAnalysePhpCodeByHalsteadMetric: TBitBtn;
    lstvwOperatorInfo: TListView;
    lstvwOperandInfo: TListView;
    btnDeleteComments: TButton;

    procedure OpenPhpFileExecute(Sender: TObject);
    procedure ClosePhpFileExecute(Sender: TObject);
    procedure FormCreate(Sender: TObject);
    procedure btnAnalysePhpCodeByHalsteadMetricClick(Sender: TObject);
    procedure btnDeleteCommentsClick(Sender: TObject);

  private
    { Private declarations }
  public
    { Public declarations }
  end;

var
  MainFrm: TMainFrm;


implementation

{$R *.dfm}

const
  MaxOperators = 200;
  MaxOperands = 50;
  CountOfOperators = 47;
  CountOfKeyWords = 66;

  Key_Words: array[1..CountOfKeyWords] of string = ('__halt_compiler', 'abstract', 'and', 'array', 'as', 'break',
  'callable', 'case', 'catch', 'class', 'clone', 'const', 'continue', 'declare',
  'default', 'die', 'do', 'echo', 'else', 'elseif', 'empty', 'enddeclare',
  'endfor', 'endforeach', 'endif', 'endswitch', 'endwhile', 'eval', 'exit', 'extends',
  'final', 'for', 'foreach', 'function', 'global', 'goto', 'if', 'implements', 'include',
  'include_once', 'instanceof', 'insteadof', 'interface', 'isset', 'list', 'namespace',
  'new', 'or', 'print', 'private', 'protected', 'public', 'require', 'require_once',
  'return', 'static', 'switch', 'throw', 'trait', 'try', 'unset', 'use', 'var', 'while', 'xor', 'yield');

  predefined_constants: array[1..8] of string = ('__CLASS__', '__DIR__', '__FILE__', '__FUNCTION__', '__LINE__',
  '__METHOD__', '__NAMESPACE__', '__TRAIT__');

  Operators: array[1..CountOfOperators] of string = ('===','<<=', '>>=', '!==', '++', '--', '==', '!=', '->', '^=', '.=', '&=', '**', '||', '&&',
  '<<', '>>', '<>', '<=', '>=', '?:', '|=', '%=', '/=', '*=', '+=', '-=', '(', '<', '>', '=', '+', '-', ',', '.', ';', '[]', '{',
  '}', '*', '/', '$', '@', '!', '|', '~', '&');

  alphabet: array[1..66] of string = ('a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q',
   'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '_', '.', '-', '>',
   'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q',
   'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z');

  SingleQuote = #39;
  DoubleQuote = '"';
  OperandSymbol = '$';
  OpeningTag = '<?php';
  ClosingTag = '?>';

type
  TOperators = record
    operator: string;
    count_of_inclusions: word;
  end;

  TOperands = record
    Operand: string;
    count_of_inclusions: word;
  end;

  TAllOperands = array[1..MaxOperands] of TOperands;
  TAllOperators = array[1..MaxOperators] of TOperators;

function GetString(var strIndex: integer): string; forward;
procedure CleanCodeFromCommentaries(); forward;
procedure DeleteQuotesFromText(Quotes: string); forward;
procedure DeleteQuotesFromString(Quotes: string; i: integer); forward;
procedure DeleteMultiLineComment(const comment: string); forward;
procedure GetCommentCoornidates(comment: string; i: integer; var StartLine: integer; var FinishLine: integer; var IndexOfFirstCommentOccurrence: integer; var IndexOfSecondCommentOccurrence: integer); forward;
procedure SaveChangesInText(Str: string; NumberLine: integer); forward;
procedure DeleteSingleLineComment(const comment: string); forward;
procedure SaveResultToFile(var outFile: TextFile; var OperatorsList: TAllOperators; var OperandsList: TAllOperands); forward;
procedure AnalyseTextByHalsteadMetric(var OperatorsList: TAllOperators; var OperandsList: TAllOperands); forward;
procedure CheckForOperands(var OperandsList: TAllOperands); forward;
procedure CheckOperandsOccurrence(var str: string; j: integer; var OperandsList: TAllOperands); forward;
function isSignOk(sign: string): boolean; forward;
function isExistInListOfOperands(const str: string; var OperandsList: TAllOperands): Boolean; forward;
procedure AddToOperandList(const Operand: string; var OperandsList: TAllOperands); forward;
procedure UpdateOperandList(const str: string; var OperandsList: TAllOperands); forward;
procedure AddToListOperators(var OperatorsList: TAllOperators); forward;
procedure AddToListOperands(var OperandsList: TAllOperands); forward;
procedure FreeList(var info: TAllOperators); forward;
procedure CheckTextForOperator(const Operator: string; var OperatorsList: TAllOperators); forward;
procedure CheckPresenceOfSubstring(const Operator: string; var str: string; i: integer; var OperatorsList: TAllOperators); forward;
function ExistInListOfOperators(const str: string; var OperatorsList: TAllOperators): Boolean; forward;
procedure AddToOperatorList(const Operator: string; var OperatorsList: TAllOperators); forward;
procedure UpdateOperatorList(const str: string; var OperatorsList: TAllOperators); forward;
procedure FindPhpTags(); forward;
function CheckPhpTags(Tag: string; str: string): boolean;  forward;


procedure TMainFrm.FormCreate(Sender: TObject);
begin
  MainFrm.mmoInputText.Text:= '';
  MainFrm.btnAnalysePhpCodeByHalsteadMetric.Enabled:= false;
end;

procedure TMainFrm.btnDeleteCommentsClick(Sender: TObject);
begin
  CleanCodeFromCommentaries;
end;

procedure FindPhpTags();
var
  i: integer;
  isOpenTag: boolean;
  str: string;
begin
  isOpenTag:= false;
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
  begin
    str:= MainFrm.mmoInputText.Lines.Strings[i];
    if (isOpenTag = true) then
      if (CheckPhpTags(ClosingTag, str) = true) then
        isOpenTag:= false;
    if ((CheckPhpTags(OpeningTag, str) = false) and (isOpenTag = false)) then
    begin
      delete(str, 1, Length(str));
      SaveChangesInText(str, i);
    end
    else
    begin
      isopenTag:= true;
      if (CheckPhpTags(OpeningTag, str) = true) then
        delete(str, 1, Length(str));
      SaveChangesInText(str, i);
    end;
  end;
end;

function CheckPhpTags(Tag: string; str: string): boolean;
var
  TagPos: integer;
begin
  TagPos:= pos(Tag, str);
  if ((TagPos <> 0) and (TagPos = 1)) then
    result:= true
  else
    result:= false;
end;

procedure CleanCodeFromCommentaries();
begin
  DeleteQuotesFromText(DoubleQuote);
  FindPhpTags();
  DeleteMultilineComment('/*');
  DeleteSingleLineComment('//');
  DeleteSingleLineComment('#');
end;

procedure DeleteQuotesFromText(Quotes: string);
var
  i: integer;
begin
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
    DeleteQuotesFromString(Quotes, i);
end;

procedure DeleteQuotesFromString(Quotes: string; i: integer);
var
  IndexOfFirstCommentOccurrence, IndexOfSecondCommentOccurrence, isExistOtherQuotes: integer;
  str: string;
begin
  str:= MainFrm.mmoInputText.Lines.Strings[i];
  IndexOfFirstCommentOccurrence:= Pos(Quotes, str);
  if (IndexOfFirstCommentOccurrence <> 0) then
  begin
    delete(str, IndexOfFirstCommentOccurrence, 1);
    SaveChangesInText(str, i);
  end;
  IndexOfSecondCommentOccurrence:= Pos(Quotes, str);
  if (IndexOfSecondCommentOccurrence <> 0) then
  begin
    delete(str, IndexOfFirstCommentOccurrence, (IndexOfSecondCommentOccurrence - IndexOfFirstCommentOccurrence + 1));
    SaveChangesInText(str, i);
  end;
  isExistOtherQuotes:= Pos(Quotes, str);
  if (isExistOtherQuotes <> 0) then
    DeleteQuotesFromString(Quotes, i);
end;

function GetString(var strIndex: integer): string;
var
  str: string;
begin
  repeat
    str := MainFrm.mmoInputText.Lines.Strings[strIndex];
    Inc(strIndex);
  until (strIndex > MainFrm.mmoInputText.Lines.Count) or (str <> '');
  if (strIndex > MainFrm.mmoInputText.Lines.Count) then
    str := '';
  result:= str;
end;

procedure DeleteMultiLineComment(const comment: string);
var
  i, j, IndexOfFirstCommentOccurrence, IndexOfSecondCommentOccurrence, StartLine, FinishLine: integer;
  str: string;
begin
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
  begin
    GetCommentCoornidates(comment, i, StartLine, FinishLine, IndexOfFirstCommentOccurrence, IndexOfSecondCommentOccurrence);
    if ((IndexOfFirstCommentOccurrence <> 0) and (IndexOfSecondCommentOccurrence <> 0)) then

      if StartLine <> FinishLine then
      begin
        str:= MainFrm.mmoInputText.Lines.Strings[i];
        delete(str, IndexOfFirstCommentOccurrence, (length(str) - IndexOfFirstCommentOccurrence));
        SaveChangesInText(str, StartLine);
        for j:= (StartLine + 1) to FinishLine do
        begin
          if j = FinishLine then
          begin
            delete(str, 1, IndexOfSecondCommentOccurrence);
            SaveChangesInText(str, FinishLine);
          end
          else
          begin
            MainFrm.mmoInputText.Lines.Delete(i);
            SaveChangesInText(str, FinishLine);
          end;
        end;
      end
      else
      begin
        delete(str, IndexOfFirstCommentOccurrence, IndexOfSecondCommentOccurrence - IndexOfFirstCommentOccurrence);
        SaveChangesInText(str, StartLine);
      end;

  end;
end;

procedure GetCommentCoornidates(comment: string; i: integer; var StartLine: integer; var FinishLine: integer; var IndexOfFirstCommentOccurrence: integer; var IndexOfSecondCommentOccurrence: integer);
var
  j: integer;
  str: string;
begin
  str:= GetString(i);
  IndexOfFirstCommentOccurrence:= Pos(Comment, str);
  if (IndexOfFirstCommentOccurrence <> 0) then
  begin
    StartLine:= i;
    for j:= i to (MainFrm.mmoInputText.Lines.Count - 1) do
    begin
      str:= MainFrm.mmoInputText.Lines.Strings[j];
      IndexOfSecondCommentOccurrence:= Pos('*/', str);
      if (IndexOfSecondCommentOccurrence <> 0) then
      begin
        FinishLine:= j;
        break;
      end;
    end;
  end;
end;

procedure SaveChangesInText(Str: string; NumberLine: integer);
begin
   MainFrm.mmoInputText.Lines.Strings[NumberLine]:= str;
end;

procedure DeleteSingleLineComment(const comment: string);
var
  i, IndexOfFirstCommentOccurrence: integer;
  str: string;
begin
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
  begin
    str:= MainFrm.mmoInputText.Lines.Strings[i];
    IndexOfFirstCommentOccurrence:= Pos(Comment, str);
    if (IndexOfFirstCommentOccurrence <> 0) then
      delete(str, IndexOfFirstCommentOccurrence, (length(str)));
    SaveChangesInText(str, i);
  end;
end;

procedure TMainFrm.ClosePhpFileExecute(Sender: TObject);
begin
  MainFrm.Close;
end;

procedure TMainFrm.OpenPhpFileExecute(Sender: TObject);
begin
  if MainFrm.DlgOpen.Execute then
    try
      MainFrm.mmoInputText.Lines.LoadFromFile(DlgOpen.FileName);
      MainFrm.btnAnalysePhpCodeByHalsteadMetric.Enabled:= true;
    except
      ShowMessage('Íå óäàëîñü îòêðûòü ôàéë.');
    end;
end;

procedure TMainFrm.btnAnalysePhpCodeByHalsteadMetricClick(Sender: TObject);
var
  outFile: TextFile;
  OperatorsList: TAllOperators;
  OperandsList: TAllOperands;
begin
  freeList(OperatorsList);
  if DlgSave.Execute then
    try
      AssignFile(outFile, DlgSave.FileName);
      Rewrite(outFile);
      AnalyseTextByHalsteadMetric(OperatorsList, OperandsList);
      SaveResultToFile(outFile, OperatorsList, OperandsList);
      CloseFile(outFile);
    except
      ShowMessage('Íå óäàëîñü îòêðûòü ôàéë.');
    end;
end;

procedure SaveResultToFile(var outFile: TextFile; var OperatorsList: TAllOperators; var OperandsList: TAllOperands);
var
  i, n, n1, n2, ProgramLenght, TotalSumOperands, TotalSumOperators: integer;
  V: real;
begin
  n1:= 0;
  TotalSumOperators:= 0;
  for i:= 1 to MaxOperators do
    if (OperatorsList[i].count_of_inclusions <> 0) then
    begin
      n1:= n1 + OperatorsList[i].count_of_inclusions;
      inc(TotalSumOperators);
    end;
  n2:= 0;
  TotalSumOperands:= 0;
  for i:= 1 to MaxOperands do
    if (OperandsList[i].count_of_inclusions <> 0) then
    begin
      n2:= n2 + OperandsList[i].count_of_inclusions;
      inc(TotalSumOperands);
    end;
  n:= n1 + n2;
  ProgramLenght:= TotalSumOperators + TotalSumOperands;
  V:= ProgramLenght*Log2(n);
  writeln(OutFile, 'Ñëîâàðü ïðîãðàììû n = ', n);
  writeln(OutFile, 'Äëèíà ïðîãðàììû N = ', ProgramLenght);
  writeln(OutFile, 'Îáúåì ïðîãðàììû V = ', V:7:3);
end;

procedure AnalyseTextByHalsteadMetric(var OperatorsList: TAllOperators; var OperandsList: TAllOperands);
var
  i: integer;
begin
  CheckForOperands(OperandsList);
  for i:= 1 to CountOfOperators do
    CheckTextForOperator(Operators[i], OperatorsList);
  for i:= 1 to CountOfKeyWords do
    CheckTextForOperator(Key_Words[i], OperatorsList);
  AddToListOperators(OperatorsList);
  AddToListOperands(OperandsList);
end;

procedure CheckForOperands(var OperandsList: TAllOperands);
var
  i: integer;
  str: string;
begin
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
  begin
    str:= MainFrm.mmoInputText.Lines.Strings[i];
    CheckOperandsOccurrence(str, i, OperandsList);
  end;
end;

procedure CheckOperandsOccurrence(var str: string; j: integer; var OperandsList: TAllOperands);
var
  i, IndexOfFirstOccurrence, IndexOperandsEnd, OperandsLength: integer;
  operand: string;
begin
  IndexOperandsEnd:= length(str);
  IndexOfFirstOccurrence:= Pos(OperandSymbol, str);
  if (IndexOfFirstOccurrence <> 0) then
  begin
    for i:= (IndexOfFirstOccurrence + 1) to length(str) do
      if (isSignOk(str[i]) = true) then
      begin
        IndexOperandsEnd:= i;
        break;
      end;
    OperandsLength:= IndexOperandsEnd - IndexOfFirstOccurrence;
    operand:= copy(str, IndexOfFirstOccurrence + 1, OperandsLength - 1);
    delete(str, IndexOfFirstOccurrence, OperandsLength);
    if (not isExistInListOfOperands(Operand, OperandsList)) then
      AddToOperandList(Operand, OperandsList)
    else
      UpdateOperandList(Operand, OperandsList);
    SaveChangesInText(str, j);
    CheckOperandsOccurrence(str, j, OperandsList);
    MainFrm.mmoInputText.Lines.Strings[j]:= str;
  end;
end;

function isSignOk(sign: string): boolean;
var
  i: integer;
begin
  for i:= 1 to 66 do
  begin
    if (sign = alphabet[i]) then
    begin
      result:= false;
      break;
    end
    else
      result:= true;
  end;
end;

function isExistInListOfOperands(const str: string; var OperandsList: TAllOperands): Boolean;
var
  i: integer;
begin
  Result := False;
  for i:= 1 to MaxOperands do
  begin
    if (OperandsList[i].operand = str) then
    begin
      Result := true;
      break;
    end;
  end;
end;

procedure AddToOperandList(const Operand: string; var OperandsList: TAllOperands);
var
  i: integer;
begin
  for i:= 1 to MaxOperands do
  begin
    if (OperandsList[i].operand = '') and (OperandsList[i].count_of_inclusions = 0) then
    begin
      OperandsList[i].operand:= Operand;
      OperandsList[i].count_of_inclusions:= 1;
      break;
    end;
  end;
end;

procedure UpdateOperandList(const str: string; var OperandsList: TAllOperands);
var
  i: integer;
begin
  for i:= 1 to MaxOperands do
  begin
    if (OperandsList[i].operand = str) then
    begin
      inc(OperandsList[i].count_of_inclusions);
      break;
    end;
  end;
end;

procedure AddToListOperators(var OperatorsList: TAllOperators);
var
  i: integer;
begin
  for i:= 1 to MaxOperators do
  begin
    if (OperatorsList[i].operator <> '') then
    begin
      with MainFrm.lstvwOperatorInfo.Items.Add do
      begin
        Caption := inttostr(index);
        SubItems.Add(OperatorsList[i].operator);
        SubItems.Add(IntToStr(OperatorsList[i].count_of_inclusions));
      end;
    end;
  end;
end;

procedure AddToListOperands(var OperandsList: TAllOperands);
var
  i: integer;
begin
  for i:= 1 to MaxOperands do
  begin
    if (OperandsList[i].operand <> '') then
    begin
      with MainFrm.lstvwOperandInfo.Items.Add do
      begin
        Caption := inttostr(index);
        SubItems.Add(OperandsList[i].operand);
        SubItems.Add(IntToStr(OperandsList[i].count_of_inclusions));
      end;
    end;
  end;
end;

procedure FreeList(var info: TAllOperators);
var
  i: integer;
begin
  for i:= 1 to MaxOperators do
  begin
    info[i].operator:= '';
    info[i].count_of_inclusions:= 0;
  end;
end;

procedure CheckTextForOperator(const Operator: string; var OperatorsList: TAllOperators);
var
  i: integer;
  str: string;
begin
  for i:= 0 to (MainFrm.mmoInputText.Lines.Count - 1) do
  begin
    str:= MainFrm.mmoInputText.Lines.Strings[i];
    CheckPresenceOfSubstring(Operator, str, i, OperatorsList);
  end;
end;

procedure CheckPresenceOfSubstring(const Operator: string; var str: string; i: integer; var OperatorsList: TAllOperators);
var
  IndexOfFirstOccurrence: integer;
begin
  IndexOfFirstOccurrence:= Pos(Operator, str);
  if (IndexOfFirstOccurrence <> 0) then
  begin
    delete(str, IndexOfFirstOccurrence, length(Operator));
    if (not ExistInListOfOperators(Operator, OperatorsList)) then
      AddToOperatorList(Operator, OperatorsList)
    else
      UpdateOperatorList(Operator, OperatorsList);
    CheckPresenceOfSubstring(Operator, str, i, OperatorsList);
    MainFrm.mmoInputText.Lines.Strings[i]:= str;
  end;
end;

function ExistInListOfOperators(const str: string; var OperatorsList: TAllOperators): Boolean;
var
  i: integer;
begin
  Result := False;
  for i:= 1 to MaxOperators do
  begin
    if (OperatorsList[i].operator = str) then
    begin
      Result := true;
      break;
    end;
  end;
end;

procedure AddToOperatorList(const Operator: string; var OperatorsList: TAllOperators);
var
  i: integer;
begin
  for i:= 1 to MaxOperators do
  begin
    if (OperatorsList[i].operator = '') and (OperatorsList[i].count_of_inclusions = 0) then
    begin
      OperatorsList[i].operator:= Operator;
      OperatorsList[i].count_of_inclusions:= 1;
      break;
    end;
  end;
end;

procedure UpdateOperatorList(const str: string; var OperatorsList: TAllOperators);
var
  i: integer;
begin
  for i:= 1 to MaxOperators do
  begin
    if (OperatorsList[i].operator = str) then
    begin
      inc(OperatorsList[i].count_of_inclusions);
      break;
    end;
  end;
end;

end.
