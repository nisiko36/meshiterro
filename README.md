# README

This README would normally document whatever steps are necessary to get the
application up and running.

Things you may want to cover:

* Ruby version

* System dependencies

* Configuration

* Database creation

* Database initialization

* How to run the test suite

* Services (job queues, cache servers, search engines, etc.)

* Deployment instructions

* ...


























import {
  Container,
  Heading,
  FormControl,
  FormLabel,
  Text,
  Button,
  Textarea,
  FormErrorMessage,
  Modal,
  ModalOverlay,
  ModalContent,
  ModalHeader,
  ModalCloseButton,
  ModalBody,
  ModalFooter,
  useDisclosure,
  Table,
  Thead,
  Tbody,
  Tr,
  Th,
  Td,
} from "@chakra-ui/react";
import { useForm, Controller } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";
import { useContext, useState, useEffect } from "react";
import { ResultContext } from "../context/ResultContext";
import { ocrFunction } from "../api/ocr";
import { gptFunction } from "../api/gpt";

// インターフェース: フォームの型定義
interface UploadForm {
  file: FileList;
  prompt: string;
  tuning: string;
}

interface Template {
  prompt: string;
  tuning: string;
}

// バリデーションスキーマ
const uploadSchema = z.object({
  file: z
    .instanceof(FileList)
    .refine((files) => files.length > 0, { message: "ファイルを選択してください。" })
    .refine(
      (files) => files[0]?.type === "application/pdf",
      { message: "PDFファイルをアップロードしてください。" }
    ),
  prompt: z
    .string()
    .min(1, { message: "プロンプトは1文字以上必要です。" })
    .max(2000, { message: "プロンプトは2000文字以下にしてください。" }),
  tuning: z
    .string()
    .refine(
      (value) => {
        try {
          JSON.parse(value);
          return true;
        } catch {
          return false;
        }
      },
      { message: "チューニングパラメータは有効なJSON形式である必要があります。" }
    ),
});

const templateSchema = z.object({
  prompt: z.string().min(1, "プロンプトは必須です"),
  tuning: z.string().refine(
    (value) => {
      try {
        JSON.parse(value);
        return true;
      } catch {
        return false;
      }
    },
    { message: "チューニングパラメータは有効なJSON形式である必要があります" }
  ),
});

export const UploadPage = () => {
  const { setResultData } = useContext(ResultContext);
  const { control, handleSubmit, formState: { errors }, register } = useForm<UploadForm>({
    resolver: zodResolver(uploadSchema),
  });

  const { isOpen, onOpen, onClose } = useDisclosure();
  const [templates, setTemplates] = useState<Template[]>([]);
  const [newTemplate, setNewTemplate] = useState<Template>({ prompt: "", tuning: "" });
  const [editTemplate, setEditTemplate] = useState<Template | null>(null);
  const [isCreateMode, setIsCreateMode] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null);

  // ページが読み込まれたときにローカルストレージからデータを取得
  useEffect(() => {
    const storedTemplates = localStorage.getItem("templates");
    if (storedTemplates) {
      setTemplates(JSON.parse(storedTemplates));
    }
  }, []);

  // テンプレートをローカルストレージに保存する関数
  const saveTemplates = (newTemplates: Template[]) => {
    setTemplates(newTemplates);
    localStorage.setItem("templates", JSON.stringify(newTemplates));
  };

  // 新規テンプレート作成モーダル
  const handleTemplateCreate = () => {
    const parsedTemplate = templateSchema.safeParse(newTemplate);
    if (!parsedTemplate.success) {
      console.log(parsedTemplate.error);
      return;
    }
    const updatedTemplates = [...templates, newTemplate];
    saveTemplates(updatedTemplates);
    setNewTemplate({ prompt: "", tuning: "" });
    onClose();
  };

  // 編集を保存する関数
  const handleEditSave = () => {
    if (selectedIndex !== null && editTemplate) {
      const parsedTemplate = templateSchema.safeParse(editTemplate);
      if (!parsedTemplate.success) {
        console.log(parsedTemplate.error);
        return;
      }
      const updatedTemplates = [...templates];
      updatedTemplates[selectedIndex] = editTemplate;
      saveTemplates(updatedTemplates);
      onClose();
    }
  };

  // テンプレートを削除する関数
  const handleTemplateDelete = (index: number) => {
    const updatedTemplates = templates.filter((_, i) => i !== index);
    saveTemplates(updatedTemplates);
  };

  const myClickFunction = async (data: UploadForm) => {
    try {
      setResultData(null);
      const ocrResult = await ocrFunction(data.file[0]);
      const gptResult = await gptFunction(ocrResult, data.prompt, data.tuning);
      setResultData(gptResult);
      console.log('GPTの結果:', gptResult);
    } catch (error) {
      console.error("APIの呼び出しに失敗しました", error);
    }
  };

  return (
    <Container mt={12}>
      <Heading as={"h2"} mb={8}>
        ファイルアップロード
      </Heading>
      <form onSubmit={handleSubmit(myClickFunction)}>
        {/* ファイルアップロード */}
        <div>
          <label htmlFor="file">ファイルをアップロードしてください (PDF)</label>
          <input
            type="file"
            id="file"
            {...register('file')}
          />
          {errors.file && <p style={{ color: 'red' }}>{errors.file.message}</p>}
        </div>

        {/* プロンプト入力 */}
        <FormControl isInvalid={!!errors.prompt} mt={8}>
          <FormLabel htmlFor="prompt">プロンプト</FormLabel>
          <Controller
            name="prompt"
            control={control}
            render={({ field }) => (
              <Textarea
                {...field}
                id="prompt"
                placeholder="プロンプトを入力してください"
              />
            )}
          />
          <FormErrorMessage>{errors.prompt && errors.prompt.message}</FormErrorMessage>
        </FormControl>

        {/* チューニングパラメータ */}
        <FormControl isInvalid={!!errors.tuning} mt={8}>
          <FormLabel htmlFor="tuning">チューニングパラメータ (JSON)</FormLabel>
          <Controller
            name="tuning"
            control={control}
            render={({ field }) => (
              <Textarea
                {...field}
                id="tuning"
                placeholder="チューニングパラメータを入力してください"
              />
            )}
          />
          <FormErrorMessage>{errors.tuning && errors.tuning.message}</FormErrorMessage>
        </FormControl>

        <Button type="submit" mt={8} colorScheme="teal">
          アップロード
        </Button>
      </form>

      {/* テンプレートモーダル */}
      <Button onClick={onOpen} mt={8} colorScheme="blue">
        テンプレート
      </Button>

      <Modal isOpen={isOpen} onClose={onClose}>
        <ModalOverlay />
        <ModalContent>
          <ModalHeader>{isCreateMode ? "テンプレート作成" : "テンプレート編集"}</ModalHeader>
          <ModalCloseButton />
          <ModalBody>
            <FormControl>
              <FormLabel>プロンプト</FormLabel>
              <Textarea
                value={isCreateMode ? newTemplate.prompt : editTemplate?.prompt || ""}
                onChange={(e) =>
                  isCreateMode
                    ? setNewTemplate({ ...newTemplate, prompt: e.target.value })
                    : setEditTemplate({ ...editTemplate!, prompt: e.target.value })
                }
              />
            </FormControl>

            <FormControl mt={4}>
              <FormLabel>チューニングパラメータ</FormLabel>
              <Textarea
                value={isCreateMode ? newTemplate.tuning : editTemplate?.tuning || ""}
                onChange={(e) =>
                  isCreateMode
                    ? setNewTemplate({ ...newTemplate, tuning: e.target.value })
                    : setEditTemplate({ ...editTemplate!, tuning: e.target.value })
                }
              />
            </FormControl>
          </ModalBody>
          <ModalFooter>
            <Button onClick={onClose} mr={3}>
              閉じる
            </Button>
            <Button colorScheme="teal" onClick={isCreateMode ? handleTemplateCreate : handleEditSave}>
              {isCreateMode ? "作成" : "保存"}
            </Button>
          </ModalFooter>
        </ModalContent>
      </Modal>

      {/* テンプレート一覧 */}
      <Table mt={8}>
        <Thead>
          <Tr>
            <Th>インデックス</Th>
            <Th>プロンプト</Th>
            <Th>チューニングパラメータ</Th>
            <Th>編集</Th>
            <Th>削除</Th>
          </Tr>
        </Thead>
        <Tbody>
          {templates.map((template, index) => (
            <Tr key={index}>
              <Td>{index + 1}</Td>
              <Td>{template.prompt}</Td>
              <Td>{template.tuning}</Td>
              <Td>
                <Button
                  colorScheme="yellow"
                  size="sm"
                  onClick={() => {
                    setSelectedIndex(index);
                    setEditTemplate(template);
                    setIsCreateMode(false);
                    onOpen();
                  }}
                >
                  編集
                </Button>
              </Td>
              <Td>
                <Button colorScheme="red" size="sm" onClick={() => handleTemplateDelete(index)}>
                  削除
                </Button>
              </Td>
            </Tr>
          ))}
        </Tbody>
      </Table>
    </Container>
  );
};
